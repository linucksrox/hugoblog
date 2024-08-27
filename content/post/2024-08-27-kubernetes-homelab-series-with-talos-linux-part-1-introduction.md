---
layout: blog
draft: true
title: Kubernetes Homelab Series With Talos Linux Part 1 - Introduction
date: 2024-08-27T23:06:24.328Z
tags:
  - kubernetes
  - homelab
summary: This in depth series will walk through building a Kubernetes cluster
  far beyond the basics, including dynamically provisioned storage, certificate
  management, and backups.
---
# Introduction

W﻿e are going to cover building a Kubernetes cluster from the ground up, suitable for a homelab or production use. I will go into much greater depth than a lot of tutorials do, well beyond "the basics."

I﻿ use Proxmox in my homelab, but this could be followed easily in VMWare or other hypervisors of choice.

B﻿uilding a cluster is easy. Making it work for you is not. As many problems as Kubernetes solves, it introduces many new challenges and ways of thinking about things. One of the hardest problems is storage, which we will talk about and I will demonstrate how to do this in a homelab. It can be done with a single disk. You don't have to use Rancher Longhorn. And you don't even have to use NFS. We'll be looking at NFS and NVMe-oF and how to actually get them working.

O﻿ther topics will include LoadBalancer services using MetalLB, secret management using Sealed Secrets (and also talking about SOPS with age), cluster backup of the etcd cluster along with Velero to back up cluster resources.

F﻿inally, and this will actually be the first topic, I will be using Talos Linux because I believe it's the easiest option overall considering the initial build and also ongoing maintenance such as OS upgrades and Kubernetes version upgrades.

# Prerequisites and Assumptions

* Y﻿ou have a homelab of some sort and can download ISOs and create VMs
* Y﻿ou have a disk or virtual disk to dedicate for storage (or you can do clustered storage, we'll talk about it)
* .﻿.. I think that's it? For now at least.