---
layout: blog
draft: false
title: GitLab Runners On EKS Using Cluster Autoscaler
date: 2026-02-18
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
1. No shared image cache - job startup time was abysmal due to a combination of insane image sizes and basically no reuse of cached images (due to new EC2 instances for every job)
1. Base AMI images needed to be updated manually when the security team asked (bad, we should obviously be automating and doing this proactively, but nobody had time or desire to prioritize this)
1. docker+machine sometimes got in a weird state where it could not scale up new EC2 instances and instead would get stuck in a retry loop, preventing any new jobs in that queue from running until we resolved it manually
1. docker+machine has been deprecated for several years - we tried updating to a later version but things broke. It wasn't worth the burden of tracking down why, rather than looking for a better path forward.
1. Observability. We just don't have a good way to measure performance or monitor for issues (though admittedly that is largely an "us" issue)
1. Long lived EC2 instances. In our configuration, docker+machine idles down to 1 node per queue. In practice, this means that the first ever EC2 instance that was deployed for that queue remains online as long as possible, storing up image cache and other ephemeral data that is unneeded. Even with 200GB disks, we've had many occasions where it was necessary to SSH into the EC2 instances and run prune commands. We tried automating this but something about `docker system prune -af` non-interactively is unreliable over an SSH session.
1. Troubleshooting is tricky. We can SSH into the host running docker+machine, get a list of EC2 instances, then `docker-machine ssh instance-name` to access the instance and troubleshoot.
1. Cost (and waste). One EC2 instance per job adds up, especially when the vast majority of those jobs use under 10% of the allocated resources.

This project kind developed organically out of multiple needs that seemed to come in all around the same time. I was already bored with the idea of the existing maintenance plan, and we were not proactive about it partly due to the toil involved. Developers started complaining about slow startup time - this was because of the lack of shared image caching, combined with developers pulling multi gigabyte images for test jobs. Sure, it's easy to say they should optimize their images (and it couldn't be more true). Security started pushing on us to "patch" the runners, which meant updating the base AMI images manually, but if we needed to keep this up to date monthly, we were going to need to automate somewhere. Now there was a compelling reason for me to get buy-in from leadership to assign this project, and I already had a plan in mind for how we could replace docker+machine and simplify the maintenance burden - Kubernetes. I had hopes that it would solve every pain point we had with docker+machine:
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
I'm not new to Kubernetes, but I was new to EKS. I wanted to use a GitLab pipeline to build the EKS infrastructure. After some quick research, I found `eksctl` to be the easiest way to build a cluster. It simplifies many aspects that would otherwise require a fair bit of Cloudformation, but is flexible **enough** to get the job done, with only minor effort required outside of that.

Other options included Cloudformation or CDK, but why bother when there is a bespoke tool directly from Amazon.

I built a pipeline in GitLab to deploy EKS using `eksctl` and AWS CLI. It relies on a YAML file that is specific to `eksctl` to define some specifics that I needed. I needed to use the AWS CLI to change the EKS upgrade policy to STANDARD. The pipeline is idempotent, so that I can safely run it as many times as I want without making unnecessary changes (think "if cluster exists, then `eksctl upgrade cluster` else `eksctl create cluster`"). I built some logic around nodegroups, so that when I change or add nodegroups they are deployed properly, but existing nodegroups are not changed or removed. I also opted to use access entries for authentication, so by including the name of the role in the config file, the pipeline automatically ensures the proper access entry is added for that role. That allows us to authenticate with the control plane through our AWS SSO configuration.

## Networking
Originally, I was leaning toward putting everything in its own VPC. However, there was additional work to get the GitLab VPC talking to the EKS VPC, but most importantly I quickly realized that some jobs were connecting to external systems over the public internet with IP allow listing. I needed to reuse the existing EIP if at all possible.

I chose to place the EKS workers in the same VPC subnet as the existing runners due to IP allow listing on external systems from the NAT gateway EIP. It was easier if all outbound traffic came from the same IP. Don't get me started on why we send all of this traffic over the public internet to begin with, that's a battle for another day.

One gotcha about the AWS CNI that comes out of the box is that assigns a unique IP from your VPC subnet to every pod. I was not expecting this, and during initial POC testing I quickly found out when I started scaling up parallel jobs. Luckily it's possible to add a new CIDR block to the existing VPC, so I went ahead and added a new block and split that into 2 subnets in different AZs, then wired those up to the existing route table and out through the existing NAT gateway. Crisis averted. We could have switched CNIs as well, but I was ready to move forward, and since this was supported it was a little bit less effort in the short term. If we start going beyond 1000 parallel jobs then I'll be revisiting this one and moving to something else. For now, we're well below that, so it's noted and we can move on.

## Compute
The other primary decision to make was compute. EKS delivers the control plane, but you still need somewhere to run your stuff. The two options are EC2 or Fargate (we don't talk about hybrid nodes). For this project's goal, EC2 was the obvious choice because spinning up a single EC2 instance instantly provides capacity for X number of jobs to run in parallel. Maybe even more importantly is that jobs within the EC2 instance can share the image cache, greatly increasing the odds that a given job will start significantly faster if its target image is already cached.
