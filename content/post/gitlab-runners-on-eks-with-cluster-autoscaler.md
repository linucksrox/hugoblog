---
layout: blog
draft: true
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

# The Build
I'm not new to Kubernetes, but I was new to EKS. 