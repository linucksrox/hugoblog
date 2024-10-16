---
layout: blog
draft: true
title: Kubernetes Homelab Series Part 7 - Backups With Velero
date: 2024-08-28T13:44:32.722Z
tags:
  - kubernetes
  - homelab
  - backup
  - etcd
summary: A look into backing up etcd in a Talos Linux cluster, along with full
  cluster resource backups using Velero
---
# The Importance Of Backups
It's easy to gloss over backups especially when you are setting up a new cluster, finally getting stuff running on it, and feeling excited that you can now deploy some service and basically automate certificates, ingress, and everything else that used to be a pain to do manually. But as soon as something fails, you will wish you spent a little more time not only thinking about backups, but also documenting step by step how to recover from a disaster.

Backups in Kubernetes are EASY. Just do it now, before you put stuff on the cluster, and write down the steps to recover. Why not?

# etcd Backups In Talos Linux
First we need to talk about etcd backups. What is etcd (I think it's pronounced "et-see-dee")? This is the database that the control plane nodes use to keep track of any and all resources in the cluster. It's the database of all the manifests, with stuff like deployments, replicasets, secrets, pods, and metadata that is currently running in the cluster.

Side note: This tracks real time data so whatever storage the etcd cluster is running on (the actual disks the control plane node stores this data on) needs to be pretty fast. If it's slow, you can run into issues where `kubectl` commands are very slow, or stuff stops working as expected in the cluster. Maybe this note belongs in the Talos Linux setup section, but here we are.

Talos Linux, unlike a traditional kubeadm install, does not run etcd as a static pod. Instead, it runs on the node behind the Talos API, meaning you have to use `talosctl` to manage it.

## How To Back Up etcd In Talos
- (commands to back up)
You could automate this somewhere so you have regular backups. Highly recommended :)

## How To Recover etcd In Talos
- (commands to recover)

# What's Velero?
https://velero.io/
Velero is a backup tool that allows you to not only back up everything, but also back up specific types of resources, along with scheduling snapshots, etc. It then allows you to recover specific resources, even from a full backup, etc. So it is significantly more flexible than taking full etcd backups (but it's still recommended to take etcd backups as well, for DR). Velero is a useful tool for Kubernetes cluster migrations.

# Velero Installation/Setup
f
