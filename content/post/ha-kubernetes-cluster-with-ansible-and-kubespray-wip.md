---
layout: blog
draft: false
title: HA Kubernetes Cluster With Ansible and Kubespray (WIP)
date: 2022-02-11T15:11:35.321Z
summary: This is an overview of how to build an HA Kubernetes cluster using
  Ansible with Kubespray for reproducible results.
thumbnail: /images/pepperspray.jpg
feature_image: /images/pepperspray.jpg
tags:
  - kubernetes
  - ansible
---
# This is currently a work in progress, mostly for my own benefit to track my progress through this build. There may be major steps missing!

## Prerequisites

As a starting point, you need 6 VMs which are accessible via SSH so that Ansible can remote in and make all the necessary changes. VM provisioning and Ansible/SSH keys are beyond the scope of this guide but are a prerequisite. I currently run Proxmox and use Terraform to generate the fresh Ubuntu VMs and inject my Ansible SSH key so that it can do its thing. I strongly recommend using a tool like Terraform that allows you to quickly destroy and recreate fresh VMs since you will need to do this process any time the Kubespray deployment fails or you want to make big changes and deploy again.

* Create 6 fresh Ubuntu based VMs to be used for 3 masters and 3 workers (using Terraform for this step)
* Install SSH keys into the VMs so that Ansible can remote in with sudo privileges (using Terraform for this step)
* On your Ansible machine, `git clone` the kubespray repo
* Build your inventory file for Ansible to connect into the VMs it will be modifying
* Make changes as needed to your Kubespray deployment.
* Optionally but strongly recommended, track any customizations in your own git repo. There are some different approaches to this but I'm currently using a private repo and tracking the official kubespray repo as an upstream branch.
* I wanted to use kube-vip for HA on the control plane API which is currently not built into Kubespray, so I created a new playbook that copies a yaml file into the static pod manifest folder which deploys kube-vip for the control plane nodes. I run this playbook which does that task and then calls the kubespray cluster.yml playbook afterward.
* Run the Kubespray playbook. Something like this: `ansible-playbook -i inventory/mycluster/hosts.yaml -b kube-vip-cluster.yml -u ubuntu`
* Since I couldn't seem to get kube-vip working as a LoadBalancer service, I'm using MetalLB for that. Deploy MetalLB and don't forget to add the config which tells it which IP address pool to use.

  * You can just enable MetalLB in Kubespray, but I opted to deploy using the official manifest in order to decouple it from the core cluster creation. I want to keep the cluster as barebones as possible for deployment, then install all add-ons separately, eventually using GitOps with something like ArgoCD.
* Install an ingress controller such as Traefik.

  * This requires a ClusterRole, ClusterRoleBinding, ServiceAccount, Daemonset or Deployment, and a Service of type LoadBalancer.
* Install Rancher. There are several options for managing clusters such as Lens or K9s. After using Lens for a little while I find it difficult to switch between client machines, especially when you are destroying and recreating a cluster for testing which means you have to reconnect manually over and over. I like the idea of a web based management interface because it is more centralized, and Rancher does the job nicely.
* Install a distributed block storage solution. I am currently testing Longhorn, but ran into an issue with the RWX mode when deploying bitnami/wordpress from the helm chart ([Longhorn issue #3661](https://github.com/longhorn/longhorn/issues/3661))
* Install a monitoring solution. I haven't spent enough time with the ELK stack or many other monitoring solutions but that is one of the next steps on my radar.
* Install a centralized logging solution. I will evaluate Graylog and Logstash/Kibana.
* Install applications
  * Migrate existing Nextcloud VM instance to Kubernetes
