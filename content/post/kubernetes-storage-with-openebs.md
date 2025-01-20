---
layout: blog
draft: false
title: Kubernetes Storage - OpenEBS Replicated Storage Mayastor (WIP)
date: 2025-01-20
tags:
  - kubernetes
  - homelab
  - storage
  - openebs
  - mayastor
summary: Let's walk through deploying OpenEBS Replicated Storage with the Mayastor engine on Talos Linux!
---
# Intro and Prerequisites
In a previous post, I mentioned that I struggled to get OpenEBS working in Talos and instead went with democratic-csi. In recent weeks, I decided to revisit this and figure out how to get OpenEBS replicated storage working in order to evaluate replicated storage in my cluster. I now have multiple disks that I can dedicate to my Kubernetes cluster and wanted to avoid the issue with the single point of failure using a TrueNAS VM for democratic-csi.

If you are following along, I will assume you are familiar with deploying Talos Linux itself and have talosctl installed with an existing cluster running. If you need more details on how to do that, check out [https://blog.dalydays.com/post/kubernetes-homelab-series-part-1-talos-linux-proxmox/](https://blog.dalydays.com/post/kubernetes-homelab-series-part-1-talos-linux-proxmox/).

# Dedicated Storage Node
It's not absolutely necessary to use a dedicated storage node. I'm doing this because I want to pass a disk directly to a VM for storage on each physical host and want to keep storage somewhat isolated from other worker nodes, and I can spare the few extra resources to dedicate to this purpose. If you want to use existing worker nodes, just follow this process for your existing nodes instead of creating new ones.

## Create New Talos Node
- Create a VM in Proxmox with 2GB RAM and 2 CPU cores. I named my first one talos-storage-1
  - Attach a Talos ISO to the CD ROM and boot from it
  - Get the IP address from the node
- Install Talos using the worker.yaml template used for other worker nodes (you may want to get a current or updated version of Talos from the image factory):
  - `talosctl apply-config --insecure -n 10.0.50.135 --file _out/worker.yaml`
- Apply a patch to set a static IP and node label, e.g.
```yaml
# ./patches/storage1.patch
machine:
  network:
    hostname: talos-storage-1
    interfaces:
      - deviceSelector:
          busPath: "0*"
        addresses:
          - 10.0.50.31/24
        routes:
          - network: 0.0.0.0/0
            gateway: 10.0.50.1
    nameservers:
      - 192.168.1.22
```
  - `talosctl patch mc -n 10.0.50.135 --patch @patches/storage1.patch`
- Apply a patch to set some machine config stuff for OpenEBS, like hugepages and specific bind mounts:
```yaml
# ./patches/openebs.patch
machine:
  sysctls:
    vm.nr_hugepages: "1024"
  nodeLabels:
    openebs.io/engine: mayastor
  kubelet:
    extraMounts:
      - destination: /var/local
        type: bind
        source: /var/local
        options:
          - rbind
          - rshared
          - rw
      - destination: /var/openebs/local
        type: bind
        source: /var/openebs/local
        options:
          - rbind
          - rshared
          - rw
```
  - `talosctl patch mc -n 10.0.50.31 --patch @patches/openebs.patch`