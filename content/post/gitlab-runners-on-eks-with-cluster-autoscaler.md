---
layout: blog
draft: false
title: GitLab Runners On EKS Using Cluster Autoscaler
date: 2026-02-20
tags:
  - kubernetes
  - eks
  - autoscaling
  - gitlab
summary: A deep dive on using EKS to power GitLab runners with autoscaling, and how I overcame some gotchas.
---
# Background
Inheriting legacy infrastructure is an adventure. We had been using the legacy runners for many years now, before I started with the company. They had some issues but overall they served their purpose. They were based on docker+machine which allowed for dynamically scaling EC2 instances in AWS. Those runners checked all the boxes originally - they ran jobs, they autoscaled, and they were easy to configure when needed. It was a great starting point, maybe even the best available option at the time. But over the years the pain points continued to grow, and I was already passively considering alternatives. Some of the pain points include:
1. One job per EC2 instance - I was disappointed when I found this out, to say the least
1. Slow scale up - it would take about 5 minutes for a new EC2 instance to bootstrap before it could accept a job
1. docker+machine weirdness - it would orphan EC2 instances forever, requiring periodic manual cleanup
1. No shared image cache - job startup time was abysmal due to a combination of insane image sizes and essentially no reuse of cached images (due to new EC2 instances for every job)
1. Base AMI images needed to be updated manually when the security team asked (bad, we should obviously be automating and doing this proactively, but nobody had time or desire to prioritize this)
1. docker+machine sometimes got in a weird state where it could not scale up new EC2 instances and instead would get stuck in a retry loop, preventing any new jobs in that queue from running until we resolved it manually
1. docker+machine has been deprecated for several years - we tried updating to a later version but things broke. It wasn't worth the burden of tracking down why, rather than looking for a better path forward.
1. Observability. We just don't have a good way to measure performance or monitor for issues (though admittedly that is largely an "us" issue)
1. Long lived EC2 instances. In our configuration, docker+machine idles down to 1 node per queue. In practice, this means that the first ever EC2 instance that was deployed for that queue remains online as long as possible, storing up image cache and other ephemeral data that is unneeded. Even with 200GB disks, we've had many occasions where it was necessary to SSH into the EC2 instances and run prune commands. We tried automating this but something about `docker system prune -af` non-interactively is unreliable over an SSH session.
1. Troubleshooting is tricky. We can SSH into the host running docker+machine, get a list of EC2 instances, then `docker-machine ssh instance-name` to access the instance and troubleshoot.
1. Cost (and waste). One EC2 instance per job adds up, especially when the vast majority of those jobs use under 10% of the allocated resources.

This project kind of developed organically out of multiple needs that seemed to come in all around the same time. I was already bored with the idea of the existing maintenance plan, and we were not proactive about it partly due to the toil involved. Developers started complaining about slow startup time - this was because of the lack of shared image caching, combined with developers pulling multi gigabyte images for test jobs. Sure, it's easy to say they should optimize their images (and it couldn't be more true). Security started pushing on us to "patch" the runners, which meant updating the base AMI images manually, but if we needed to keep this up to date monthly, we were going to need to automate somewhere. Now there was a compelling reason for me to get buy-in from leadership to assign this project, and I already had a plan in mind for how we could replace docker+machine and simplify the maintenance burden - Kubernetes. I had hopes that it would solve every pain point we had with docker+machine:
1. Workers could run multiple jobs, making better use of available resources. After all, that was supposed to be one of the main benefits of using Docker.
1. Fast scale up. Larger EC2 instances per worker meant that adding one more EC2 instance allowed X more jobs to run as soon as the new instance was ready, plus we could use lightweight base images rather than our bulky AMI.
1. Cluster Autoscaler should be capable of handling scaling in production environments, and it is actively maintained unlike docker+machine.
1. Better shared image cache. Since many jobs would now run per EC2 instance, all of those jobs would share the image cache that was pulled to that instance. There are better ways, but this was still a good start, and significantly better than the existing option.
1. Using AWS provided AMIs meant we no longer have to maintain or build custom AMIs with security and logging agents, etc.
1. Ideally, eliminate docker+machine weirdness by simply eliminating docker+machine itself.
1. EKS is supported by AWS, along with their AMIs. Kubernetes is actively maintained. Cluster Autoscaler is actively maintained. GitLab runners support Kubernetes.
1. We can get insight into overall health and performance using EKS container insights. It's a major leap forward from what we had before, although not the most impressive tool in the world either.
1. No more long-lived EC2 instances. Cluster autoscaler happily scales down any which one, and given even a low amount of activity throughout the days and weeks it does a nice job balancing instances and cycling them out, which I love to see. I don't need stale workers starting to exhibit weird issues that I probably don't care about the root cause.
1. Maybe this one is personal preference, but I find it much easier to troubleshoot within Kubernetes. I use `k9s` which makes it amazingly easy (dare I say fun) to see the state of all pods running in the namespace, exec into them if desired, and easily get a picture of the pods, nodes, resource utilization, logs, etc. in realtime.
1. Thanks to the magic of Cluster Autoscaler, we can idle down the worker nodes very, very low while there are no jobs demanding resources. On average, I estimated this to save at least 50%, assuming the exact same workload we ran last year, with potential to save even more if we continue monitoring and tuning. And this still includes all the other benefits including less maintenance and faster performance.

Our team also explored a simpler option where we could build and maintain a few EC2 instances running Docker and use the docker executor. This would be a lot more in line with what the original engineer thought was going to happen when he decided on docker+machine years ago, running more than 1 job per EC2 instance. This would solve some of our pain points, but still leave us with some maintenance burden, having to care about long-lived EC2 instances, and still having to patch or automate a rolling build. It would provide better scaling and resource utilization, but still require us to manually right-size forever. While the team built and benchmarked a POC on this idea, I was testing on EKS. We figured we knew which option was going to win, but of course you don't know until you try.

It turns out that yes, EKS is better in almost every way. It really did solve the pain points, got us into a much better security posture, and reduced our maintenance burden by well over 50%. Job startup time increased on average, and job performance even increased significantly. I may elaborate on that piece later on.

It was also the perfect way to introduce a production Kubernetes environment to the organization (yes this is the first thing we have ever done as a company with Kubernetes) because of the relative level of complexity.

# Design
This was the goal: to build a scalable, maintainable platform on which to run stateless jobs for GitLab as efficiently as possible. I was solving for all of the pain points listed above, but primarily thinking about job startup performance, security, and long term maintenance. Cost savings is always good, but for this project it was just the icing on the cake.

## Build Tooling
I'm not new to Kubernetes, but I was new to EKS. I wanted to use a GitLab pipeline to build the EKS infrastructure. After some quick research, I found `eksctl` to be the easiest way to build a cluster. It simplifies many aspects that would otherwise require a fair bit of CloudFormation, but is flexible **enough** to get the job done, with only minor effort required outside of that.

Other options included CloudFormation or CDK, but why bother when there is a bespoke tool directly from Amazon.

I built a pipeline in GitLab to deploy EKS using `eksctl` and AWS CLI. It relies on a YAML file that is specific to `eksctl` to define some specifics that I needed. I needed to use the AWS CLI to change the EKS upgrade policy to STANDARD. The pipeline is idempotent, so that I can safely run it as many times as I want without making unnecessary changes (think "if cluster exists, then `eksctl upgrade cluster` else `eksctl create cluster`"). I built some logic around nodegroups, so that when I change or add nodegroups they are deployed properly, but existing nodegroups are not changed or removed. I also opted to use access entries for authentication, so by including the name of the role in the config file, the pipeline automatically ensures the proper access entry is added for that role. That allows us to authenticate with the control plane through our AWS SSO configuration.

## Networking
Originally, I was leaning toward putting everything in its own VPC. However, there was additional work to get the GitLab VPC talking to the EKS VPC, but most importantly I quickly realized that some jobs were connecting to external systems over the public internet with IP allow listing. I needed to reuse the existing EIP if at all possible.

I chose to place the EKS workers in the same VPC subnet as the existing runners due to IP allow listing on external systems from the NAT gateway EIP. It was easier if all outbound traffic came from the same IP. Don't get me started on why we send all of this traffic over the public internet to begin with, that's a battle for another day.

## Compute
The other primary decision to make was compute. EKS delivers the control plane, but you still need somewhere to run your stuff. Our two options are EC2 or Fargate.

Based on this project's goal, EC2 was the obvious choice because spinning up a single EC2 instance instantly provides capacity for X number of jobs to run in parallel. Maybe even more importantly is that jobs within the EC2 instance can share the image cache, greatly increasing the odds that a given job will start significantly faster if its target image is already cached.

Within EC2, I had 3 options: bring your own EC2, managed nodegroups, or EKS Auto Mode. At the time, Auto mode was so new that I wasn't ready to commit to that for production workloads. Perhaps that would have turned out great, but it wasn't ready in my opinion. Bring your own has its own obvious downsides, and we have another option so it was the obvious choice to go with managed nodegroups. It's yet another thing that simplifies some of the deployment and maintenance burden, and we generally trust AWS solutions to work well which has been the case with managed nodegroups for us.

## Access Control
I'm talking about specifically how to authenticate to the cluster API. It was a no-brainer to map IAM roles that our team already uses with SSO to Access Entries on the cluster. This allows us to go into the terminal, run `assume` (a granted.dev tool) and authenticate to AWS SSO.

I evaluated IRSA, but there was more effort to integrate with our OIDC provider and we didn't have a need for fine-grained access controls up front.

RBAC is an area where we have lots of room for improvement. For now, it's like giving the whole team root access to the whole cluster. Granted it's only our team who has any access, but we still don't follow the principle of least privilege here at this time, partly due to the process requiring manual updates via Helm, etc. No other teams require any level of access to the cluster, so we aren't concerned with taking a tiered approach in this particular situation.

## Config Management
Let's talk about updating `config.toml`. I originally included the GitLab runner config within the same project repo that builds and deploys the EKS cluster. I now realize my mistake, because changing a runner config (maybe we need to increase the log limit) means running the EKS deployment pipeline. While it is idempotent, it's still awkward at best. Also, the process is to update the repo, run through the pipeline, and on top of that we still have a manual process to authenticate and manually run `helm` commands to upgrade deployments.

I see a couple of opportunities for improvement:
- I should have put gitlab runner jobs in their own namespace. I did create a namespace for everything gitlab runner related, but this also includes the gitlab-runner deployment itself which is a minor annoyance, and doesn't help when aggregating things like performance metrics.
- Config for gitlab-runner deployments (used with the Helm charts) are mixed into the same repo as the EKS infrastructure build. This should go in its own repo.
- GitOps. What was I thinking? I should have used FluxCD up front. Even fixing some issues with config management and splitting out namespaces properly leaves us with a significant amount of manual effort to make simple config changes.
  - The process is more tedious than I would expect. With the old system it was: SSH into the test GitLab host > backup and live edit `config.toml` > test > repeat for production. With the new system, the process is longer: checkout a feature branch > update the config > manually test the config by running `helm upgrade` commands (assuming I'm already set up with kubectl, kubeconfig, authentication, helm, helm repos, etc.) > test > merge to main > repeat for production.
  - Why bother updating a config in a repo when you're going to change it manually in production? We all hope nobody ever makes a mistake and they get out of sync, but I've never seen this work well, anywhere. I assume there will ALWAYS be drift because we built a system that allows that there CAN be drift. What if there's some production outage, and the on-call needs to fix it quickly overnight? The chances it will go back into the repo later are low, despite everyone's best intentions.
  - All this complexity, all a waste of time and effort. What if there was a system that could proactively sync what's in the config repo with what's currently running in the cluster? Welcome to GitOps.
    - This is on the roadmap before we do anything new in the cluster. The new process will be: update the config in the repo > open a MR > test (in the cluster) > merge to main > validate in production
    - Anyone can do it
    - Nobody needs to install special tools or question IAM permissions, including junior engineers or even interns
    - We can enforce change approvals
    - Visibility - we have an audit trail. If it's not in the repo, it's not in the cluster
  - I haven't even talked about what else is deployed in the cluster beyond gitlab runners. It's not much, but it's still something. FluxCD can and should also manage Cluster Autoscaler and Metrics Server.
  - FluxCD flips the traditional deployment model on its head, which has major security implications. If we grant FluxCD enough privileges to do its job, then we no longer need every engineer on our team to have full root access to the cluster via kubectl commands (any changes need to go through the repo, through FluxCD). Let's go into read only mode for debugging/troubleshooting but leave updates/writes to GitOps.
    - Sure, we still need a way in, maybe a break-glass option. One simple approach would be to authenticate to the AWS account where EKS lives, manually add a new access entry, and boom, you're in. Document this. Or even automate this, put a manual job in the repo pipeline called "break glass" with clear, simple instructions. It should be easy for anyone to find and execute, but also auditable.

## All Together
I've talked through most of the design, much of which came from the initial POC but some of which comes from having been running this in production for over a year and what I've learned as new needs arise.

This also doesn't give the full picture of everything involved in making it work. Beyond deploying EKS, managed nodegroups, cluster autoscaler, metrics server, and gaining access to the cluster, we still have to deploy gitlab-runners and wire them up to gitlab to accept jobs. We also have to consider nuances when it comes to how cluster autoscaler works and some quirks that it has.

Deploying this was a very low risk deployment due to the ability to add the new EKS infrastructure in parallel with existing runners, allowing us to phase out the old stuff tag by tag. EKS runners have new, unique tags, and eventually we would add legacy tags, and finally phase out the old runners once we validated that jobs worked reliably.

In the end it has been a success, despite having some snags along the way. We learned new things, and always came up with a new, better way to move forward. One of the major benefits to this architecture is the flexibility. It feels unlimited, there doesn't seem to be any hurdle too big to get past, and most hurdles have been fairly minor.

Other factors I considered throughout this process:
- Skillset. Our team is not experienced in Kubernetes, though several on the team are working towards the CKA certification. But with a team of Linux Admins, I know everyone is capable of following the steps. Once you get past the initial hurdle of getting authenticated and setting up local tools, the rest is pretty straightforward even for inexperienced engineers.
  - The way I see it, this is a good way to level up our team in general. In many ways already, it demonstrates best practices and the way things could/should be. It gives everyone production experience with Kubernetes and this only makes us better.
- Unknown behavior with migrating all of the different pipeline jobs to the new system. In theory it should work the same because it has the same default CPU/RAM resources available. In practice, resource limits are handled very differently. We are essentially introducing new limits or enforcing stricter limits than what we had previously. And this is essential so that we can tune scaling, establish safe defaults, and manage cost over time.
- New maintenance schedule and process. EKS upgrades. Add-on upgrades. Component upgrades. Nodegroup AMI upgrades. The great news here is that it's possible to do all of these while workloads continue running, so we just unlocked significant maintenance improvements and security by extension by being able to keep things updated more frequently with less friction. And we can patch zero-day vulnerabilities any time with very little risk.
- Migration from legacy runners. The plan was to introduce the new runners and ask developers to start testing. We had mixed results. The backup plan was to start switching tags to the new runners, and then removing those tags from the old runners, phasing them out. Once we phase the old runners out and haven't had any outages or break/fix tickets come in, we can finally decommission the old infrastructure and fully realize the security and cost savings.

# Gotchas
## > eksctl
This is a handy utility, but is not idempotent by design. If you deploy a new cluster with this tool, and later on decide to change something about that cluster (but don't want to rebuild), you can't necessarily update your config and redeploy. Even the command is different: new builds are `eksctl create cluster` where certain changes can be done with `eksctl upgrade cluster`. Some changes to managed nodegroups can be done with `eksctl update nodegroup` but others require completely rebuilding the nodegroup. But in the end, the total cluster config and deployment pipeline is significantly simpler to build and understand than it would have been using CloudFormation.

## > EKS Built-in CNI
The AWS CNI that comes out of the box is that assigns a unique IP from your VPC subnet to every pod. I was not expecting this, and during initial POC testing I quickly found out when I started scaling up parallel jobs. Luckily it's possible to add a new CIDR block to the existing VPC, so I went ahead and added a new block and split that into 2 subnets in different AZs, then wired those up to the existing route table and out through the existing NAT gateway. Crisis averted. We could have switched CNIs as well, but I was ready to move forward, and since this was supported it was a little bit less effort in the short term. If we start going beyond 1000 parallel jobs then I'll be revisiting this one and moving to something else. For now, we're well below that, so it's noted and we can move on.

## > Manual Changes
Manual changes still need to be made in the current state. After building the cluster (fully automated), it's in a vanilla state and requires you to manually deploy your resources. For us this includes Metrics Server, Cluster Autoscaler and gitlab-runners. This is an area where GitOps would help tremendously.

## > Spot Instances
This isn't too difficult. In the eksctl config, enable `spot: true` and be sure to specify at least 2 or 3 different instance types within the same family with the same minimum CPU/RAM specs. Just in case one type is not available during spot request, it can pick from something else so that jobs are not stuck pending. I also ran into this. You're already saving so much money, this is not the time to be a stickler.

I think that's it for spot instances.

## > OOMKilled Jobs
In our case going from EC2 instances with 8GB RAM and 4GB swap to pods with a strict 8GB limit caused a couple of OOMKills here and there. It turns out this was one of the unknowns we anticipated going into the migration.

Why does this happen if the RAM limit matches? On the EC2 instance, the job has full reign of the VM it runs on, so not only is the Linux kernel more conservative about preventing OOMkills, it also has a 4GB swap buffer that it can exhaust before giving up.

This is actually a major factors I found to be related to the significant performance increase for some workloads on EKS, despite no changes to the pipeline and similar resource limites from a CPU/RAM perspective. Why would a job run 25% faster? It could be that we were silently exceeding the resource limits on legacy runners but it wasn't bad enough to break jobs, just bad enough to make them slow.

Does this mean EKS is worse or less reliable in a way? No. This makes it obvious that jobs were already overprovisioned, but nobody was aware of it. Nobody was even aware of the performance degradation due to heavy over-utilization of swap. I see it as a feature, it highlights the need for better visibility, and it highlights the importance of understanding what we **truly** need from a resource perspective to run jobs. No more throwing hardware at the problem.

We should understand what jobs cost from a resource and financial perspective, because that will allow us properly tune autoscaling, make things run much more efficiently, likely save significantly on cost, and make informed decisions on resource quota increases if needed.

We're almost done with OOMkilled, but not quite. Guess what, it's super easy to override requests and limits in GitLab pipelines to ask for more CPU or RAM. I have this documented in a simple to follow guide with screenshots and specific examples of different messages that developers might see which could be related. I set upper limits on CPU and RAM requests. I explain why you should probably not set CPU limits, especially with Java workloads. And I just handed them self-service autonomy that frees up our team from troubleshooting OOMkill issues and rather provides developers what they need to keep working 24/7.

There's more to talk about on this topic, including setting lower default limits once developers are more comfortable overriding these values. Looking at utilization, 8GB RAM is a pretty high limit which means we are very loosely packing jobs and wasting a lot of resources still. Keep in mind this still fits within the 50% savings estimate, but we could be doing significantly better by tightening up requests for most jobs that only need 1-2GB, and packing more of those jobs into a single worker. Another approach to this problem is encouraging developers who will listen to override their requests for lightweight jobs to lower values, proactively. Even if we leave the default at 8GB, if developers are proactive about this it can still offer them tangible benefits. They are more likely to get their jobs scheduled sooner, because a 2GB slot is easier to schedule than an 8GB slot, etc.

## > Cluster Autoscaler
To me, Cluster Autoscaler is the source of the most interesting gotchas I ran into. These took the most time and caused most of the hurdles that we have had to overcome. Here's what I ran into, in order.

### >> Scaling Up
This is the easy part. Deploy Cluster Autoscaler (CA) and configure its min/max thresholds. However, there's a little bit of glue you need to tie this together with the Managed Nodegroups and the AutoScalingGroups (ASG) they manage. It's not 100% ready out of the box. Here's a recap of what needs to happen, how it works:
- CA needs permission to read and make changes to the ASG. This is done via an IAM role attached to the EC2 instance powering your worker nodes (every worker node has the same profile). If you specify `autoScaler: true` in the eksctl config, this is all handled automatically.
- Cluster Autoscaler (CA) interacts directly with the ASG, increasing desired capacity to scale up. It identifies the correct ASG by matching with tags on the ASG. These are already set in the eksctl config.
- It's pretty easy to check if this works. If you submit more jobs from GitLab, wait a minute and then check to see if the ASG requested more instances. If that doesn't work, move to troubleshooting (checking CA logs, IAM permissions, etc.).

### >> Scaling Down
This is where it starts to get really interesting. During initial testing, scale-up and scale-down worked great. I believe the default was 10 minutes before scaling down, but I didn't experience any problems. After running more workloads over time, I got a ticket that jobs were pending for a long time. I quickly realized that autoscaling wasn't working. The online worker nodes were taking jobs just fine, but CA wasn't doing what I expected.

What went wrong here?

The initial design was intentionally simple (KISS). I deployed a single managed nodegroup and deployed everything on it, including all gitlab-runner deployments, and CA itself. Now it's obvious in hindsight. When you run important things on the very nodes you are potentially scaling down and terminating, bad things can happen. CA terminated the node where the CA deployment itself lived, and it got stuck in a bad state.
- "If we were your kids, we'd punish ourselves." - Little Rascals, 1994

CA was punishing itself and I gave it no other choice.

The solution? What if I deployed another, smaller managed nodegroup (2 nodes for HA), labeled it management, and isolated it from CA? So that's exactly what I did. Add that to the EKS deployment, add labels and taints, and update taints/tolerations on the CA deployment. I protected it from itself, and this problem is solved.

#### > Management Nodegroup
Without going into too much detail, just keep in mind that this means we have to manage taints and tolerations so that management workloads go to management nodes, and job workloads go to job nodes. This involves node selectors and tolerations on the deployment side of things.

### >> Scaling Down To 0
This actually began as a request for a brand new type of runner we had not implemented before. There was a new need to build Dockerfiles on `arm64`, so I began evaluating options. Just because you can doesn't mean you should, and while my first option was running a QEMU based build which could spit out multi-platform images from a single `amd64` node, it wasn't efficient and wasn't worth the effort. The developers already had a completely separate Dockerfile with different build steps, so why not just give them a runner to build on `arm64` natively?

I decided to create another managed nodegroup, but given the type of workload, the infrequency, and risk if jobs get interrupted, I decided it was worthwhile to base this on spot instances, plus while we're at it let's scale this one down to 0 since most of the time it's unused and would be sitting idle, wasting money.

#### > Scaling Back Up From 0
The easy part is scaling down, that was already functioning. But Once you remove all instances from the ASG, now there's a problem. How does CA know which ASG to go modify? Previously, it would use the running Kubernetes node to discover ASG info. When you get to the point where this ASG has no worker nodes running in the cluster, CA has less visibility, requiring you to add a specific tag to the ASG that it uses to identify it.

Specifically, you need to manually add a **tag** on the ASG `k8s.io/cluster-autoscaler/node-template/label/nodegroup: [your-nodegroup-name]` where you substitute the name of your nodegroup there, plus you need to have a **label** on your managed nodegroup that matches, which can be predefined in the eksctl config when you create the nodegroup.

I may have missed how to do this. I scoured the documentation and spent some trial/error time on this, but was unable to find a solution to fully automate this tagging in the build pipeline. Therefore I have instructions to specify that whenever using a managed nodegroup that scales to 0, you need to manually add the appropriate tag on the ASG. It's one extra manual step. It would still be nice if it could be automated, and perhaps Copilot is right when it suggests it could be done by adding the right label in the right format within eksctl config.

I should revisit this some time. Even 1 simple manual step can become a larger issue over time.

### >> Scaling Down, Again
Wait, there's more?! Yes, there's more. This is the one I find the most interesting. I'm not sure if favorite is the right word but...

So we're finally in a pretty good state, and people know what to do if they see OOMKilled (they have documentation and practice adjusting limits). I receive another troubleshooting ticket. This time a job has failed because the pod was terminated unexpectedly. Let's dive into what happened.

Starting with the failed job log, it clearly states...(WIP - to be continued)

### >> AZRebalance
Here we go again. Another ticket comes in, another job terminated prematurely. I thought we fixed autoscaling (and we kind of did), but now the AZRebalance feature is biting us.

This is not exactly related to Cluster Autoscaler, but kind of similar in a way. Instead of scaling up or down, it rebalances which is a fancy way of saying add a new node in a different AZ, then terminate a node in the hot-spot AZ.

Verifying the failed job log, it failed prematurely. Digging deeper, we look at ASG events and also CloudTrail, and find a tight correlation: a node was terminated seconds after the job failed unexpectedly. Full disclosure, `kiro-cli` is an amazing tool for digging through event history and logs in AWS, and correlating data with event times. It basically did all the work in this case, although it had completely the wrong event initially, I called out the timing mismatch, and it found another AZRebalance event that matched up perfectly.

What happened in this case?
- As nodes are scaled up and down, it's possible to find yourself in a situation where there's an unbalanced number of nodes in a single AZ.
- Independently, AZRebalance is checking for this case. Its job is to rebalance when things are not balanced.
- In order to balance things, AZRebalance will spin up a new node in another AZ with fewer nodes, and then it will terminate the extras in the hot-spot AZ.
- AZRebalance doesn't discriminate, it doesn't even check what's running. It cares not about your workloads (and rightfully so, we need to build around this understanding). In this case, the node it selected as tribute was one that had production jobs running on it.

What can we do about it?
- We can add yet another tool, AWS Node Termination Handler. That also involves setting up an SQS Queue and EventBridge. Not the end of the world, but one more thing to maintain.
  - More importantly, we are under pressure to deliver on other things right now, so building this out requires more engineering effort.
- OR the quick and dirty approach is to disable AZRebalance. It's not ideal, but it got us in a more reliable state immediately. The caveat is that we document this, and make sure it's included in our backlog to resolve later on. This still leaves us in a potentially bad situation where nodes can hot-spot in a single AZ, exposing us to more risk if that AZ has an outage. But we are OK with the tradeoffs for the short term.