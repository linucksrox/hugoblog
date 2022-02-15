---
layout: blog
draft: false
title: HA Kubernetes Cluster With Ansible and Kubespray (WIP)
date: 2022-02-11T15:11:35.321Z
thumbnail: /images/pepperspray.jpg
tags:
  - kubernetes
  - ansible
summary: This is an overview of how to build an HA Kubernetes cluster using
  Ansible with Kubespray for reproducible results.
---
# This is currently a work in progress, mostly for my own benefit to track my progress through this build. There may be major steps missing!

## Prerequisites
As a starting point, you need 6 VMs which are accessible via SSH so that Ansible can remote in and make all the necessary changes. VM provisioning and Ansible/SSH keys are beyond the scope of this guide but are a prerequisite. I currently run Proxmox and use Terraform to generate the fresh Ubuntu VMs and inject my Ansible SSH key so that it can do its thing. I strongly recommend using a tool like Terraform that allows you to quickly destroy and recreate fresh VMs since you will need to do this process any time the Kubespray deployment fails or you want to make big changes and deploy again.
- Create 6 fresh Ubuntu based VMs to be used for 3 masters and 3 workers
- Install SSH keys into the VMs so that Ansible can remote in with sudo privileges
- On your Ansible machine, `git clone` the kubespray repo
- Build your inventory file for Ansible to connect into the VMs it will be modifying
- Make changes as needed to your Kubespray deployment.
- Optionally but strongly recommended, track any customizations in your own git repo. There are some different approaches to this but I'm currently using a private repo and tracking the official kubespray repo as an upstream branch.
- I wanted to use kube-vip for HA on the control plane API, so I created a new playbook that copies a yaml file into the static pod manifest folder which deploys kube-vip for the control plane nodes. So I run this playbook which does that task and then calls the kubespray cluster.yml playbook automatically.
- Run the Kubespray playbook. Something like this: `ansible-playbook -i inventory/mycluster/hosts.yaml -b kube-vip-cluster.yml -u ubuntu`
- Since I couldn't seem to get kube-vip working as a LoadBalancer service, I'm using MetalLB for that. Deploy MetalLB and don't forget to add the config which tells it which IP address pool to use.
- Install an ingress controller such as Traefik.
- I still need to discuss storage. I will be attempting to use Longhorn by Rancher because it provides distributed storage for HA. Besides deploying Longhorn or any other storage solution, I need to revise the VM setup task to include allocating some amount of dedicated storage space to the worker node VMs.
- Testing
- Deploying stuff
