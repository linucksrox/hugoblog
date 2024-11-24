---
layout: blog
draft: false
title: Kubernetes Homelab Series Part 6 - Storage With democratic-csi (WIP)
date: 2024-11-24
tags:
  - kubernetes
  - homelab
  - storage
summary: Diving into the depths of Kubernetes storage, then walking through
  using democratic-csi for NFS and NVMe-oF with Talos Linux.
---
# What's So Hard About Storage?
There are several questions to answer when deciding how to handle storage for Kubernetes.
- How many disks do you have?
- Do you want/need replicated storage?
- What are your storage capacity requirements?
- What are your performance requirements?
- Do you need dynamically provisioned storage or will you be doing it manually?
- NFS or iSCSI or NVMe-oF?
- How will you back up your data?
- Do you need snapshots?
- Does your storage need to be highly available?
- Does it need to be accessible from any node in the cluster, or are you good with node local storage?

Ultimately it's not actually hard, it's just complex if you want to achieve anything like you get with EBS volumes in AWS, but in your homelab. Here's how I always try to approach complex problems: start as simple as possible, make it work, then add complexity only as needed.

## My Requirements
- To have dynamically provisioned persistent volumes
- To have that persistent volume be accessible from any node in my cluster
- Keep it as simple as reasonably possible

## How Am I Doing It?
Currently I have a single 2TB NVMe disk and I want to use that for dynamically provisioned storage. I'm not worried about replication right now since I have backups in place and this is just for my homelab. If I wanted to do replicated storage, I might consider a Ceph cluster, but that realistically requires a decent amount of hardware and fast networking interconnectivity (greater than 1GB, ideally 10GB minimum for replication to keep up).

In order to manage the disk I'm using TrueNAS Scale, which is basically ZFS on Linux with a nice web GUI to manage things. This actually provides the option of doing a zpool with maybe 2 disks mirrored, or even a RAIDZ6 as your storage target to easily solve for replication across disks.

In Proxmox, I'm passing the disk itself directly to TrueNAS VM. You **should** pass an entire disk controller if you need to pool multiple drives together in ZFS. If you're dealing with a single disk, it's OK to do it this way.

Spin up the VM, format the disk, ZFS, etc. and you're ready to go. In my case, this is on the Kubernetes VLAN and assigned a static IP. Originally I had a second NIC attached to the primary VLAN but this caused some weird web UI performance issues so I removed that one.

# TrueNAS Setup In Proxmox
This isn't intended to be a TrueNAS tutorial, so I'll just list the steps at a high level. Basically, get a TrueNAS server running and then proceed.
- Pass a disk (or HBA controller) to a VM in Proxmox
- Install TrueNAS Scale
- Create a zpool
- Generate an API Key - in the top right corner go to Admin > API Keys
- Make sure the network is accessible from your Kubernetes cluster

# Install democratic-csi With TrueNAS
https://github.com/democratic-csi/democratic-csi

This is a straightforward CSI provider that focuses on dynamically provisioned storage from TrueNAS or generic ZFS on Linux backends. Protocols include NFS, iSCSI, and NVMe-oF. I'll show you how to use the API variation and do NFS and iSCSI shares, plus talk about almost getting NVMe-oF working.

You will install a separate Helm chart for each provisioner, and you can actually run multiple at the same time, which is what I will be doing with both NFS and iSCSI. This is helpful since NFS is great for RWX volumes (if you actually have a use case for that), while iSCSI is good for single application RWO volumes.

## Dynamic iSCSI Provisioner With freenas-api-iscsi
My single 2TB disk is in a pool named `nvme2tb`. I created a dataset in TrueNAS named `iscsi`. Those may vary in your case, so pay attention to the configuration and update those values according to your environment.

Don't forget, your Talos installation needs to include the iscsi extension or the nodes won't be able to connect to TrueNAS.

- Create a dataset named `iscsi`
- Make sure Block (iSCSI) Shares Targets is running, and click Configure
- Save the defaults for Target Global Configuration
- Add a portal on 0.0.0.0:3260 named `k8s-democratic-csi`
- Add an Initiator Group, Allow all initiators, and name it something like `k8s-talos`
- Create a Target named `donotdelete` and alias `donotdelete`, then add iSCSI group selecting the Portal and Initiator Group you just created. This prevents TrueNAS from deleting the Initiator Group if you're testing and you delete the one and only PV.
- Make note of the portal ID and the Initiator Group ID and update these values in the file `freenas-api-iscsi.yaml` if needed
  - During testing, the manually created Initiator Group was getting deleted whenever deleting the last PV. This appears to be a bug in TrueNAS somewhere according to https://github.com/democratic-csi/democratic-csi/issues/412. Essentially TrueNAS deletes the Initiator Group automatically if an associated Target is deleted and no others exist. If you followed the instructions and created a manual Target this won't be an issue :)
- Create the democratic-csi namespace: `kubectl create ns democratic-csi`
- Make that namespace privileged: `kubectl label --overwrite namespace democratic-csi pod-security.kubernetes.io/enforce=privileged`
- Create `freenas-api-iscsi.yaml` and update `apiKey`, `host`, `targetPortal`, `datasetParentName`, and `detachedSnapshotsDatasetParentName`
  ```yaml
  driver:
    config:
      driver: freenas-api-iscsi
      httpConnection:
        protocol: https
        apiKey: [your-truenas-api-key-here]
        host: 10.0.50.99
        port: 443
        allowInsecure: true
      zfs:
        datasetParentName: nvme2tb/iscsi/volumes
        detachedSnapshotsDatasetParentName: nvme2tb/iscsi/snapshots
        zvolCompression:
        zvolDedup:
        zvolEnableReservation: false
        zvolBlockSize:
      iscsi:
        targetPortal: "10.0.50.99:3260"
        targetPortals: []
        interface:
        namePrefix: csi-
        nameSuffix: "-talos"
        targetGroups:
          - targetGroupPortalGroup: 1
            targetGroupInitiatorGroup: 5
            targetGroupAuthType: None
            targetGroupAuthGroup:
        extentInsecureTpc: true
        extentXenCompat: false
        extentDisablePhysicalBlocksize: true
        extentBlocksize: 512
        extentRpm: "SSD"
        extentAvailThreshold: 0
  
  csiDriver:
    # should be globally unique for a given cluster
    name: "org.democratic-csi.freenas-api-iscsi"
  
  storageClasses:
    - name: truenas-iscsi
      defaultClass: true
      reclaimPolicy: Delete
      volumeBindingMode: Immediate
      allowVolumeExpansion: true
      parameters:
        fsType: ext4
        detachedVolumesFromSnapshots: "false"
      mountOptions: []
      secrets:
        provisioner-secret:
        controller-publish-secret:
        node-stage-secret:
        node-publish-secret:
        controller-expand-secret:
  
  volumeSnapshotClasses:
    - name: truenas-iscsi
      parameters:
        detachedSnapshots: "true"

  node:
    hostPID: true
    driver:
      extraEnv:
        - name: ISCSIADM_HOST_STRATEGY
          value: nsenter
        - name: ISCSIADM_HOST_PATH
          value: /usr/local/sbin/iscsiadm
      iscsiDirHostPath: /usr/local/etc/iscsi
      iscsiDirHostPathType: ""
  ```
- Since we are using snapshots, you also need to install a snapshot controller such as https://github.com/democratic-csi/charts/tree/master/stable/snapshot-controller
  - You can skip this step if you disable snapshots in your YAML file
  - `helm upgrade --install --namespace kube-system --create-namespace snapshot-controller democratic-csi/snapshot-controller`
  - `kubectl -n kube-system logs -f -l app=snapshot-controller`
- Deploy: `helm upgrade --install --namespace democratic-csi --values freenas-api-iscsi.yaml truenas-iscsi democratic-csi/democratic-csi`
- Verify: `kubectl get all -n democratic-csi`
  - You're looking to see that everything is fully running. It may take several minutes to spin up.

## Dynamic NFS Provisioner With freenas-api-nfs
TODO: Deploy `freenas-api-nfs` using API key and test

### Dynamic NVMe-oF Storage For Kubernetes (unfinished)
TODO: Talk about my attempt to get this working and where I left off. It works when manually mounting on a regular Linux box, but not using the provisioner. Probably an issue with my configuration, but I can't find anyone to help me debug.

# What About Other Options?

## What About Local Storage?
This fails to meet one of my main requirements of being able to mount a persistent volume to any node. The whole point of Kubernetes, at least for me, is the ability to take a node offline with no actual downtime of any services. If you have a pod connected to a PVC on a certain node, but I need to move that pod to another node, I need to reconnect to the existing persistent volume, but this method doesn't allow for that.

## What About proxmox-csi-plugin?
I also looked at the [proxmox-csi-plugin](https://github.com/sergelogvinov/proxmox-csi-plugin), but that has the same problem as with node local storage so doesn't fit my requirements.

## What About Rancher Longhorn?
Longhorn is nice and easy. It uses either NFS or iSCSI. The Talos team recommends against both NFS and iSCSI (although either can be used). I would say to use Longhorn if you need replicated storage in a homelab (like 3 disks) but don't want to figure out Ceph, etc. It's straightforward and well supported. As for downsides, I don't have firsthand experience. I hear it's great for usability, but some users complain that it's not reliable.

It's also a little more complicated to set up specifically with Talos, but Longhorn has specific instructions for installation with Talos so that shouldn't be a man blocker.

At some point I might try this out, but for now I'm sticking with the simple approach of using TrueNAS Scale with any type of zpool you want, and dynamically provisioning NFS or iSCSI using democratic-csi.

## What About Mayastor?
I tried getting this to work because the Talos team recommends it if you don't want to do a full blown Ceph cluster, etc. I was also super interested in the fact that it uses NVMe-oF which is a newer protocol (basically a modern replacement for iSCSI). I wasn't able to get it working, but I discovered a little bit about it and decided to keep it simpler for the following reasons:

- I only have a single disk currently, so I don't need any replicated storage
- It seems to be more resource intensive than democratic-csi and has more components.
- The documentation is kind of hectic. It used to be Mayastor, but now it's OpenEBS Replicated PV Mayastor or something. When deploying, it's hard to tell if I'm deploying other OpenEBS stuff I don't need or what exactly PV type I need. I think you need one type of PV to store data for one of the components even if you are ultimately trying to run the replicated PV type (mayastor) for your primary cluster storage. I don't know, it was confusing.

My failure to get this working is 100% a skill issue, but going back to my requirements I really don't need this for my homelab at this point. I may revisit this in the future.

## What About Ceph?
I would need enough disks and resources to run Ceph. It's something I really want to test out and potentially use, although probably way overkill for my homelab. I'm currently running 2 Minisforum MS-01 servers and planning on getting a third to do a true Proxmox cluster and replace my old 2U power hungry server. At that point, I might actually give Ceph a shot (trying both the Proxmox Ceph installation and also Rook/Ceph on top of Kubernetes). This would solve for truly HA storage, plus meet all other requirements I have, assuming I don't add a requirement for low resource utilization just to run storage.