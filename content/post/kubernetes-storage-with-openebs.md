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
- Create a VM in Proxmox with 4GB RAM and 4 vCPU cores (2GB RAM is not enough due to the fact that you will be enabling hugepages which takes up 2GB and you would see oom-kills otherwise. You also need a dedicated CPU core just for the io-engine, along with all the other stuff that runs. I tried with 2vCPU and it wouldn't schedule the io-engine pod due to insufficient resources). I named my first one talos-storage-1
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
This was a pain to figure out. Documentation from OpenEBS is lacking, and so is documentation from Talos on the same topic. Here's what I found to work. You need a privileged namespace, bind mounts on all worker nodes, then DiskPools before you can start testing PVCs.

## Bind Mounts
After deploying the chart initially, lots of the pods were stuck initializing. It turns out that OpenEBS runs a 3 replica etcd statefulset which could run on any worker node, and that means you need to add the `/var/local` bind mount to every worker node, not only your storage nodes.

Let's start with that:
```yaml
# ./patches/bind-var-local.patch
machine:
  kubelet:
    extraMounts:
      - destination: /var/local
        type: bind
        source: /var/local
        options:
          - rbind
          - rshared
          - rw
```
- I have 3 workers in addition to my storage nodes so I ran these
  - `talosctl patch mc -n 10.0.50.21 --patch @pathces/bind-var-local.patch`
  - `talosctl patch mc -n 10.0.50.22 --patch @pathces/bind-var-local.patch`
  - `talosctl patch mc -n 10.0.50.23 --patch @pathces/bind-var-local.patch`

## Privileged Namespace
OpenEBS requires privileges, and the easiest way to handle that is by making the namespace privileged (rather than messing with machine configs).

- Add a new privileged namespace. The Helm chart wants you to use `openebs` so do this:
```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: openebs
  labels:
    pod-security.kubernetes.io/enforce: privileged
    pod-security.kubernetes.io/warn: privileged
    pod-security.kubernetes.io/audit: privileged
```
- `kubectl apply -f namespace.yaml`

## Helm Installation
- `helm repo add openebs https://openebs.github.io/openebs`
- `helm repo update`
- Grab the values from the Helm chart (`helm show values openebs/openebs > values.yaml`), or use this. I have already modified the config to disable initContainers which is a known issue with Talos, and disabled local provisioners that I'm not interested in.
```yaml
# values.yaml
openebs-crds:
  csi:
    volumeSnapshots:
      enabled: true
      keep: true

# Refer to https://github.com/openebs/dynamic-localpv-provisioner/blob/v4.1.2/deploy/helm/charts/values.yaml for complete set of values.
localpv-provisioner:
  rbac:
    create: true

# Refer to https://github.com/openebs/zfs-localpv/blob/v2.6.2/deploy/helm/charts/values.yaml for complete set of values.
zfs-localpv:
  crds:
    zfsLocalPv:
      enabled: true
    csi:
      volumeSnapshots:
        enabled: false

# Refer to https://github.com/openebs/lvm-localpv/blob/lvm-localpv-1.6.2/deploy/helm/charts/values.yaml for complete set of values.
lvm-localpv:
  crds:
    lvmLocalPv:
      enabled: true
    csi:
      volumeSnapshots:
        enabled: false

# Refer to https://github.com/openebs/mayastor-extensions/blob/v2.7.2/chart/values.yaml for complete set of values.
mayastor:
  csi:
    node:
      initContainers:
        enabled: false
  etcd:
    # -- Kubernetes Cluster Domain
    clusterDomain: cluster.local
  localpv-provisioner:
    enabled: false
  crds:
    enabled: false

# -- Configuration options for pre-upgrade helm hook job.
preUpgradeHook:
  image:
    # -- The container image registry URL for the hook job
    registry: docker.io
    # -- The container repository for the hook job
    repo: bitnami/kubectl
    # -- The container image tag for the hook job
    tag: "1.25.15"
    # -- The imagePullPolicy for the container
    pullPolicy: IfNotPresent

engines:
  local:
    lvm:
      enabled: false
    zfs:
      enabled: false
  replicated:
    mayastor:
      enabled: true
```
- `helm install openebs -n openebs openebs/openebs -f values.yaml`
- Verify: `kubectl get po -n openebs` and it should look something like this:
```
NAME                                          READY   STATUS    RESTARTS      AGE
openebs-agent-core-74d4ddc7c5-hjnxl           2/2     Running   0             9m23s
openebs-agent-ha-node-f9bsb                   1/1     Running   0             9m23s
openebs-agent-ha-node-gjdbt                   1/1     Running   0             9m23s
openebs-agent-ha-node-mwjq9                   1/1     Running   0             9m23s
openebs-agent-ha-node-rfjrw                   1/1     Running   0             93s
openebs-api-rest-757d87d4bd-zd2ms             1/1     Running   0             9m23s
openebs-csi-controller-58c7dfcd5b-6jtcq       6/6     Running   0             9m23s
openebs-csi-node-fmmwg                        2/2     Running   0             9m23s
openebs-csi-node-j95f5                        2/2     Running   2 (60s ago)   93s
openebs-csi-node-jxkvq                        2/2     Running   0             9m23s
openebs-csi-node-xtsnt                        2/2     Running   0             9m23s
openebs-etcd-0                                1/1     Running   0             9m23s
openebs-etcd-1                                1/1     Running   0             9m23s
openebs-etcd-2                                1/1     Running   0             9m23s
openebs-io-engine-lb8zr                       2/2     Running   0             9m23s
openebs-localpv-provisioner-657c44878-wjmwr   1/1     Running   0             9m23s
openebs-loki-0                                1/1     Running   0             9m23s
openebs-nats-0                                3/3     Running   0             9m23s
openebs-nats-1                                3/3     Running   0             9m23s
openebs-nats-2                                3/3     Running   0             9m23s
openebs-obs-callhome-8665bb8f6f-4ntrd         2/2     Running   0             9m23s
openebs-operator-diskpool-6d44884f8f-52rrx    1/1     Running   0             9m23s
openebs-promtail-2g6kz                        1/1     Running   0             9m23s
openebs-promtail-d72cx                        1/1     Running   0             93s
openebs-promtail-hsxlc                        1/1     Running   0             9m23s
openebs-promtail-npw7f                        1/1     Running   0             9m23s
```
- If you are seeing stuff stuck initializing, double check the pods starting with `openebs-etcd-0` since a lot of things depend on that being up before it will initialize.

## Add DiskPool(s)
I was excited at this point to test with a PVC, but then was confused about why it wouldn't provision anything. The Talos docs fall short here, seeming to imply that now is the time to test with a PVC, but if you pay close attention they only mention testing the local provisioner, adding to my confusion. It turns out you need to add DiskPools, which makes sense thinking about it. If you've ever used Longhorn there is a similar config needed after the initial install, so that you know what disk capacity you have to work with.

- Earlier, we mounted a 2TB disk to /var/mnt/nvme2tb in the talos-storage-1 node. Now we'll use that for our first DiskPool. I went with uring:
```yaml
# diskpool-1.yaml
apiVersion: "openebs.io/v1beta2"
kind: DiskPool
metadata:
  name: pool-1
  namespace: openebs
spec:
  node: talos-storage-1
  disks: ["uring:///var/mnt/nvme2tb"]

  # If you aren't running Talos, use the disk by-id path directly
  #disks: ["uring:///dev/disk/by-id/scsi-0QEMU_QEMU_HARDDISK_drive-scsi1"]
```
- `kubectl apply -f diskpool-1.yaml`
- Verify: `kubectl get dsp -n openebs`