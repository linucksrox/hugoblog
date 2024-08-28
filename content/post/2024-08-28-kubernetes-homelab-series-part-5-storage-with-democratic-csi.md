---
layout: blog
draft: true
title: Kubernetes Homelab Series Part 5 - Storage With democratic-csi
date: 2024-08-28T13:37:06.828Z
tags:
  - kubernetes
  - homelab
  - storage
summary: Diving into the depths of Kubernetes storage, then walking through
  using democratic-csi for NFS and NVMe-oF with Talos Linux.
---
# What's So Hard About Storage?
Everything. Too many questions to answer. How many disks do you have? Do you want/need replicated storage? What are your storage capacity requirements? What are your speed requirements? How many disks do you have? How fast is the networking between them if they are on separate nodes? And on and on.

Ultimately it's not actually hard, it's just complex if you want to achieve anything like you get with EBS volumes in AWS, but in your homelab. Here's how I always try to approach complex problems: start as simple as possible, make it work, then add complexity only as needed.

# How Am I Doing It?
Currently I have a single 2TB NVMe disk and I want to use that for dynamically provisioned storage. I'm not worried about replication right now since I have backups in place and this is just for my homelab. If I wanted to do replicated storage, I would be considering a Ceph cluster, but that realistically requires a decent amount of hardware and fast networking interconnectivity (greater than 1GB, ideally 10GB minimum for replication to keep up).

In order to manage the disk I'm using TrueNAS Scale, which is basically ZFS on Linux with a nice web GUI to manage things.

In Proxmox, I'm passing the disk itself directly to TrueNAS VM. You would need to pass an entire disk controller if you need to pool multiple drives together in ZFS. If you're dealing with a single disk, it's OK to do it this way.

Spin up the VM, format the disk, ZFS, etc. and you're ready to go. In my case, this is on the Kubernetes VLAN and assigned a static IP. Originally I had a second NIC attached to the primary VLAN but this caused some weird web UI performance issues so I removed that one.

# CSI Driver democratic-csi
https://github.com/democratic-csi/democratic-csi

This is a straightforward CSI provider that focuses on dynamically provisioned storage from TrueNAS or generic ZFS on Linux backends. Protocols include NFS, iSCSI, and NVMe-oF. I'm going to show you how to use the API variation and do NFS shares, along with kind of Frankenstein-ing TrueNAS Scale to also do NVMe-oF. That way you get the worst of both worlds because you have the graphical web interface, except you can't see NVMe-oF shares or manage them directly through it.

If you're strictly doing NVMe-oF you could just spin up a modern Linux server like Ubuntu and then install ZFS on Linux and use that. It's essentially the same process as the TrueNAS route for this protocol.

## Dynamic NFS Storage For Kubernetes
TODO: how to set up TrueNAS and integrate democratic-csi with the truenas-api provisioner. Also, how to test.

## Dynamic NVMe-oF Storage For Kubernetes
TODO: How to configure nvme modules on TrueNAS, how to test manually outside of Kubernetes, and how to integrate democratic-csi with it and test. This should include errors I ran into and how to troubleshoot and resolve.

# What About Rancher?
Rancher is nice and easy. It uses either NFS or iSCSI. The Talos team recommends against both NFS and iSCSI (although either can be used). I would say to use Rancher if you need replicated storage in a homelab (like 3 disks) but don't want to figure out Ceph, etc. It's straightforward and well supported. I'm not sure what the downsides are.

# What About Mayastor?
I tried getting this to work because the Talos team recommends it if you don't want to do a full blown Ceph cluster, etc. I was also super interested in the fact that it uses NVMe-oF which is a newer protocol (basically a modern replacement for iSCSI). I wasn't able to get it working, but I discovered a little bit about it and decided to keep it simpler for the following reasons:

- I only have a single disk currently, so I don't need any replicated storage
- It seems to be more resource intensive than democratic-csi and has more components.
- The documentation is kind of hectic. It used to be Mayastor, but now it's OpenEBS Replicated PV Mayastor or something. When deploying, it's hard to tell if I'm deploying other OpenEBS stuff I don't need or what exactly PV type I need. I think you need one type of PV to store data for one of the components even if you are ultimately trying to run the replicated PV type (mayastor) for your primary cluster storage. I don't know, it was confusing.