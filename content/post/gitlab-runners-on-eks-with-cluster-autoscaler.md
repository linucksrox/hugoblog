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

This project kind developed organically out of multiple needs that seemed to come in all around the same time. I was already bored with the idea of the existing maintenance plan, and we were not proactive about it partly due to the toil involved. Developers started complaining about slow startup time - this was because of the lack of shared image caching, combined with developers pulling multi gigabyte images for test jobs. Sure, it's easy to say they should optimize their images (and it couldn't be more true). Security started pushing on us to "patch" the runners, which meant updating the base AMI images manually, but if we needed to keep this up to date monthly, we were going to need to automate somewhere. Now there was a compelling reason for me to get buy-in from leadership to assign this project, and I already had a plan in mind for how we could replace docker+machine and simplify the maintenance burden - Kubernetes. I had hopes that it would solve every pain point we had with docker+machine:
1. Workers could run multiple jobs, making better use of available resources. After all, that was supposed to be one of the main benefits of using Docker.
1. Fast scale up. Larger EC2 instances per worker meant that adding one more EC2 instance allowed X more jobs to run as soon as the new instance was ready, plus we could use lightweight base images rather than our bulky AMI.
1. Cluster Autoscaler should be capable of handling scaling in production environments, and it is actively maintained unlike docker+machine.
1. Better shared image cache. Since many jobs would now run per EC2 instance, all of those jobs would share the image cache that was pulled to that instance. There are better ways, but this was still a good start, and significantly better than the existing option.
1. Using AWS provided AMIs meant we no longer have to maintain or build custom AMIs with security and logging agents, etc.
1. Ideally, eliminate docker+machine weirdness by simply eliminating docker+machine itself.
1. EKS is supported by AWS, along with their AMIs. Kubernetes is actively maintained. Cluster Autoscaler is actively maintained. GitLab runners support Kubernetes.

Our team also explored a simpler option where we could build and maintain a few EC2 instances running Docker and use the docker executor. This would be a lot more in line with what the original engineer thought was going to happen when he decided on docker+machine years ago, running more than 1 job per EC2 instance. This would solve some of our pain points, but still leave us with some maintenance burden, having to care about long-lived EC2 instances, and still having to patch or automate a rolling build. It would provide better scaling and resource utilization, but still require us to manually right-size forever. While the team built and benchmarked a POC on this idea, I was testing on EKS. We figured we knew which option was going to win, but of course you don't know until you try.

It turns out that yes, EKS is better in almost every way. It really did solve the pain points, got us into a much better security posture, and reduced our maintenance burden by well over 50%. Job startup time increased on average, and job performance even increased significantly. I may elaborate on that piece later on.

# Building
I'm not new to Kubernetes, but I was new to EKS. After some quick research, it was obvious that `eksctl` was the easiest way to build a cluster. It simplifies many aspects that would otherwise require a fair bit of Cloudformation.

I built a pipeline in GitLab to deploy EKS using `eksctl` and AWS CLI. It relies on a YAML file that is specific to `eksctl` to define some specifics that I needed. I needed to use the AWS CLI to change the EKS upgrade policy to STANDARD. The pipeline is idempotent, so that I can safely run it as many times as I want without making unnecessary changes. I built some logic around nodegroups, so that when I change or add nodegroups they are deployed properly, but existing nodegroups are not changed or removed. I also opted to use access entries for authentication, so by including the name of the role in the config file, the pipeline automatically ensures the proper access entry is added for that role. That allows us to authenticate with the control plane through our AWS SSO configuration.

For the network, the design I chose was to place the EKS workers in the same VPC subnet as the existing runners due to IP allow listing on external systems from the NAT gateway EIP. It was easier if all outbound traffic came from the same IP. And don't get me started on why we send all of this traffic over the public internet to begin with, that's a battle for another day.

One gotcha about the AWS CNI that comes out of the box is that assigns a unique IP from your VPC subnet to every pod. I was not expecting this, and during initial POC testing I quickly found out when I started scaling up parallel jobs. Luckily it's possible to add a new CIDR block to the existing VPC, so I went ahead and added a new block and split that into 2 subnets in different AZs, then wired those up to the existing route table and out through the existing NAT gateway. Crisis averted. We could have switched CNIs as well, but I was ready to move forward, and since this was supported it was a little bit less effort in the short term. If we start going beyond 1000 parallel jobs then I'll be revisiting this one and moving to something else. For now, we're well below that, so it's noted and we can move on.

