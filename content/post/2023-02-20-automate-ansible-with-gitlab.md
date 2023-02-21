---
layout: blog
draft: false
title: Automate Ansible With GitLab
date: 2023-02-20T14:22:26.973Z
feature_image: /images/screenshot-2023-02-20-at-2.00.44-pm.png
tags:
  - ansible
  - gitlab
summary: Run your Ansible playbooks directly from GitLab pipelines, allowing you
  to easily track your playbooks using IaC.
---
# My Use Case

M﻿y use case is currently to deploy new VMs to Proxmox, provision them, and finally bootstrap an RKE2 cluster in HA with 3 server nodes and 3 agents. The goal is for my pipeline to do all the heaving lifting, from cloning a template to 6 VMs, do all the provisioning such as setting static IPs and installing some packages, and then installing RKE2 and joining all nodes to the cluster. This involves a series of different playbooks that are run in different pipeline stages, all triggered from a single push. This also means that once my playbooks are stable and I don't necessarily want to push to build a new cluster, that I can manually run the pipeline from GitLab with a couple clicks to build out an entirely new cluster.

# Why GitLab CI vs. Ansible Tower/AWX?

A﻿nsible on its own is a great tool for provisioning servers and really doing any changes on a server that you used to (or still are) doing via SSH in a terminal. By moving things into an Ansible playbook, you get a more reliable, repeatable way to run through a specific set of tasks that you might want to reuse on different servers. A simple example might be installing Docker and docker compose on an Ubuntu server.

I've used Ansible Tower at work and while it's nice, I always felt like it has a weak point: certain data is not tracked in your VCS. You version control playbooks, and inventory. But then when it comes to schedules, job history, or anything else, that's all stored in Ansible Tower's database. I figure, everything that *can* be in a VCS *should* be in a VCS. This includes scheduled jobs and run history, including what variables were included when a playbook was run.

S﻿peaking of using a VCS to keep track of Ansible playbooks, GitLab is awesome. It's a bit heavy on resources for a home lab, but it really gives you all (or almost all) of the features you could want. I already have GitLab running and I already put playbooks in there. So why not run the playbooks directly from a pipeline?

T﻿his not only allows you to run a single playbook from a pipeline when pushing to a main branch, but you can also run a series of playbooks at different pipeline stages as you will see a bit later.

# What Does It Take (a.k.a. Requirements)?

You need GitLab. You need to know how a little bit about writing Ansible playbooks and creating an inventory file. You need to know a little about GitLab CI pipelines and CI/CD variables, and have GitLab already configured with at least 1 runner. You should probably at least get the basic concept of SSH keys. For this demonstration you need Proxmox. And you have to want to build a Rancher RKE2 cluster. Or just the motivation to follow along and learn for the GitLab and/or Ansible pieces.

# Show Me The Money

## Running Ansible From A Container

I﻿n order to run Ansible f﻿rom a pipeline, you just need a Python environment with Ansible installed. I always check [Docker Hub](https://hub.docker.com/) for existing (up to date) images, but surprisingly there is no official Ansible image. Maybe it's not recommended, but I can't find any reason why not. T﻿hat's just stupid. So anyway, the next best thing is to find an official, slim environment that we can install Ansible on. Since Ansible requires Python, the official Python image is perfect. I went for the python:3-slim tag since that tracks latest stable version and is very lightweight.

W﻿ith that in hand, it's just a matter of running a few steps to install ansible from pip with `python3 -m pip install --user ansible` and anything else you might want to use from your temporary container instance like openssh-client.

## GitLab Setup

S﻿etting up pipelines and runners is pretty far beyond the scope of this one, but if you're running on [GitLab.com](gitlab.com) you should be able to do this easily.

* Start with an empty repo
* Add a basic Ansible inventory file `inventory.ini`

  * I﻿ use ansible_ssh_common_args='-o StrictHostKeyChecking=no' but the alternative is to add trusted host keys into a GitLab CI/CD variable which is pretty annoying to do
* Add a playbook for testing `deploy.yml`
* Generate a new SSH key for Gitlab to connect to your Proxmox PVE host (or use one you already have)﻿
* Add a Proxmox API Token credential in Promxox and record the token ID and secret﻿﻿
* A﻿dd GitLab CI/CD variables

  * P﻿VE_API_TOKEN = the actual generated token
  * P﻿VE_API_TOKEN_ID = the name of the token user
  * P﻿VE_API_USER = root@pam (in my case)
  * P﻿VE_HOST = \[IP address of your Proxmox host]
  * S﻿SH_PRIVATE_KEY = private SSH key generated earlier (not the public key)

In the file, copypasta this. before_script and after_script will run before/after every job. This works fine for me since every job I'm running is for Ansible playbooks. So in the deploy stage, it runs before_script, then the ansible-playbook, then the after_script cleanup (so we don't leave our private key hanging out there in a container on the runner). 

{﻿{< highlight yaml >}}
image: python:3-slim

stages:
  - deploy
  - provision
  - install

before_script:
  - 'command -v ssh-agent >/dev/null || ( apt update && apt install -y openssh-client )'
  - eval $(ssh-agent -s)
  - mkdir -p ~/.ssh
  - chmod 700 ~/.ssh
  - echo "$SSH_PRIVATE_KEY" | tr -d '\r' > ~/.ssh/gitlab_ed25519
  - chmod 600 ~/.ssh/gitlab_ed25519
  - export PATH="~/.local/bin:$PATH"
  - python3 -m pip install --user ansible

after_script:
  - rm -rf ~/.ssh/

deploy:
  stage: deploy
  script:
    - ansible-playbook -i inventory.ini --user root --private-key ~/.ssh/gitlab_ed25519 -e "api_host=${PVE_HOST} api_user=${PVE_API_USER} api_token_id=${PVE_API_TOKEN_ID} api_token_secret=${PVE_API_TOKEN}" deploy.yaml
{{< /highlight >}}