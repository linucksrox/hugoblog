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

## Create New Talos Node(s)
- Create a VM in Proxmox with 3GB RAM and 2 CPU cores (2GB RAM is not enough due to the fact that you will be enabling hugepages which takes up 2GB and you would see oom-kills otherwise). I named my first one talos-storage-1
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
- Apply a patch to set some machine config stuff for OpenEBS which includes hugepages, nodeLabel and specific bind mounts:
  - Note about bind mounts: the documentation was unclear on which path to use, and I found that both were needed for different components
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
- If you have an additional disk to use with OpenEBS, you'll need to pass it directly to the Talos node VM. I'm using Proxmox
  - SSH into the Proxmox host and find the disk ID to be passed. I just run `ls -lh /dev/disk/by-id/ and get the root disk (not containing any "_1" or "_part*", for example `/dev/disk/by-id/nvme-Samsung_SSD_970_EVO_Plus_2TB_S59CNM0W635077P`)
  - Pass the disk directly to the Talos VM, where 511 is the VM ID and assuming you only have 1 disk already on scsi0: ` qm set 511 -scsi1 /dev/disk/by-id/nvme-Samsung_SSD_970_EVO_Plus_2TB_S59CNM0W635077P`
  - Checking the hardware tab on VM 511 in Proxmox, you should see this new disk. Double click it and make sure to check "Advanced", "Discard", and "SSD emulation"
    - If you see orange on these settings, you will need to shut down the VM, then power it back on for the changes to apply. Rebooting won't do it.
  - Now that the disk has been added, look for it with talosctl: `talosctl get disks -n 10.0.50.31`
    - In my case I see a disk named `sdb` which is 2.0TB with model "QEMU HARDDISK"
  - Mount the disk to a bind mount. This requires both .machine.disks and .machine.kubelet.extraMounts sections, and the mount path must start with `/var/` or it won't work.
  ```yaml
  # ./patches/mount-sdb.patch
  machine:
    kubelet:
        extraMounts:
        - destination: /var/mnt/nvme2tb
            type: bind
            source: /var/mnt/nvme2tb
            options:
            - rbind
            - rshared
            - rw
    disks:
        - device: /dev/sdb
        partitions:
            - mountpoint: /var/mnt/nvme2tb
  ```
  - Apply: `talosctl patch mc -n 10.0.50.31 --patch @patches/mount-sdb.patch` - At this point the Talos node will reboot and should come back up healthy in a minute.
  - View the console or check the dashboard with `talosctl dashbord -n 10.0.50.31`
  - If you see an error about being unable to mount the disk or the partition being the wrong type, etc. you will need to wipe the disk and create a fresh GPT partition OUTSIDE of Talos. Shut down the VM and do this in proxmox with `wipefs /dev/whatever` and then use `fdisk /dev/whatever` > `g` > `w` > `q` (`g` writes a new GPT table, `w` writes to disk and `q` quits). Now power Talos back on and it should do its thing.

Now repeat the process for any other Talos nodes you need. I have 3, so I'm doing `talos-storage-1`, `talos-storage-2` and `talos-storage-3`.

# Installing OpenEBS