---
layout: blog
draft: false
title: Automate Ansible With GitLab
date: 2023-02-20T14:22:26.973Z
feature_image: /images/screenshot-2023-02-20-at-10.09.02-am.png
tags:
  - ansible
  - gitlab
summary: Run your Ansible playbooks directly from GitLab pipelines, allowing you
  to easily track your playbooks using IaC.
---
# Use Case
M﻿y use case is currently to deploy new VMs to Proxmox, provision them, and finally bootstrap an RKE2 cluster in HA with 3 server nodes and 3 agents. The goal is for my pipeline to do all the heaving lifting, from cloning a template to 6 VMs, do all the provisioning such as setting static IPs and installing some packages, and then installing RKE2 and joining all nodes to the cluster. This involves a series of different playbooks that are run in different pipeline stages, all triggered from a single push. This also means that once my playbooks are stable and I don't necessarily want to push to build a new cluster, that I can manually run the pipeline from GitLab with a couple clicks to build out an entirely new cluster.

# Why?

A﻿nsible on its own is a great tool for provisioning servers and really doing any changes on a server that you used to (or still are) doing via SSH in a terminal. By moving things into an Ansible playbook, you get a more reliable, repeatable way to run through a specific set of tasks that you might want to reuse on different servers. A simple example might be installing Docker and docker compose on an Ubuntu server.

I've used Ansible Tower at work and while it's nice, I always felt like it has a weak point: certain data is not tracked in your VCS. You version control playbooks, and inventory. But then when it comes to schedules, job history, or anything else, that's all stored in Ansible Tower's database. I figure, everything that *can* be in a VCS *should* be in a VCS. This includes scheduled jobs and run history, including what variables were included when a playbook was run.

S﻿peaking of using a VCS to keep track of Ansible playbooks, GitLab is awesome. It's a bit heavy on resources for a home lab, but it really gives you all (or almost all) of the features you could want. I already have GitLab running and I already put playbooks in there. So why not run the playbooks directly from a pipeline?

T﻿his not only allows you to run a single playbook from a pipeline when pushing to a main branch, but you can also run a series of playbooks at different pipeline stages as you will see a bit later.

# What Does It Take?

I﻿n order to run an Ansible playbook, you just need a Python environment with Ansible installed. I always check [Docker Hub](https://hub.docker.com/) for existing (up to date) images, but surprisingly there is no official Ansible image. Maybe it's not recommended, but I can't find any reason why not. So anyway, the next best thing is to find an official, slim environment that we can install Ansible on. Since Ansible requires Python, the official Python image is perfect. I went for the python:3-slim tag since that tracks latest stable.

W﻿ith that in mind, we just need to create a GitLab pipeline, grab that image, install Ansible, do a couple other basic things like setting up SSH keys, and then you can run your playbook.