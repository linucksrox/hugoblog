---
layout: blog
draft: false
title: Kubernetes Homelab Series Part 1 - Introduction and Talos Installation
date: 2024-08-27T23:06:24.328Z
tags:
  - kubernetes
  - homelab
  - talos
summary: This in depth series will walk through building a Kubernetes cluster beyond the basics, including dynamically provisioned storage, certificate
  management, and backups.
---
# Introduction

We are going to cover building a Kubernetes cluster from the ground up, suitable for a homelab or production use (mostly). I will go into much greater depth than a lot of tutorials do, well beyond "the basics."

I use Proxmox in my homelab, but this could be followed easily in VMWare or other hypervisors of choice.

Building a bare cluster is easy. Making it work for you is not. Not because it's hard, but there are a lot of pieces and decisions to make. As many problems as Kubernetes solves, it introduces many new challenges and ways of thinking about things. One of the hardest problems is storage, which we will talk about and I will demonstrate how to do this in a homelab. Storage can be done with a single disk, and not just for hostPath. You don't have to use Rancher Longhorn. And you don't even have to use NFS. We'll be looking at NFS and NVMe-oF and how to actually get them working (for reference: https://ssdcentral.net/getting-started-with-nvme-over-fabrics-with-tcp/).

Other topics will include LoadBalancer services using MetalLB, secret management using Sealed Secrets (and also talking about SOPS with age), cluster backup of the etcd cluster along with Velero to back up cluster resources, and ingress with Traefik (plus Gateway API which is the successor to ingress controllers).

Finally, and this will actually be the first topic, I will be using Talos Linux because I believe it's the easiest option overall considering the initial build and also ongoing maintenance such as OS upgrades and Kubernetes version upgrades.

# Prerequisites and Assumptions

- You have a homelab of some sort and can download ISOs and create VMs.
- You have a disk or virtual disk to dedicate for storage (or you can do clustered storage, we'll talk about it)
- ... I think that's it? For now at least.

# Getting Started With Talos

https://www.talos.dev/
Talos Linux is Linux designed for Kubernetes – secure, immutable, and minimal.

- Supports cloud platforms, bare metal, and virtualization platforms
- All system management is done via an API. No SSH, shell or console
- Production ready: supports some of the largest Kubernetes clusters in the world
Open source project from the team at Sidero Labs

It's the easiest solution to deploy (and maintain) a Kubernetes cluster.

- Download latest Talos Linux release, grabbing metal-amd64.iso version from the GitHub releases page https://github.com/siderolabs/talos/releases
- Upload it to proxmox ISOs, or have proxmox download it directly from the URL
- Optionally rename the file manually on the filesystem: /tank/isos/template/iso/
- Create VMs in Proxmox (3 control plane nodes, 3 worker nodes)
- For OS, the talos metal-amd64 iso image. It will be replaced later during bootstrapping, so getting the latest image with added extensions now is not important.
  - Enable QEMU guest agent
  - For disks, choose something on SSD storage (required for reasonable etcd performance)
  - Enable Discard and SSD emulation options
  - 20GB should be plenty
  - Minimum of 4 CPU cores
  - Minimum of 4GB RAM
  - VLAN 50, DHCP (optional)
- Start the VMs, and check the console for each to verify they are running and have an IP assigned on the kubernetes vlan
- Check out the talos project: TBD
- Install talosctl and verify it's the same version you are deploying
- Generate new secrets which creates secrets.yaml
  - `talosctl gen secrets`
- Secure secrets.yaml with SOPS + AGE (link howto TBD)
- Create patches (elaboration TBD)
- Store talos secret somewhere (git repo)
- Apply foundational components in order
- Encrypted secret manager - sealed secrets - TBD
- LoadBalancer service - TBD
- Certificate manager - cert-manager - TBD
- CSI driver - democratic-csi - TBD
- Ingress provider - traefik - TBD
- Backup manager - velero - TBD
