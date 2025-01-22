---
layout: blog
draft: false
title: Kubernetes Storage - OpenEBS Replicated Storage Mayastor (WIP)
date: 2025-01-21
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
  - I'm having trouble here with the node name changing, and I have to manually delete the random name from the cluster:
  - e.g. `kubectl delete node talos-lry-si8`
  - Also you might need to delete the label `openebs.io/nodename=` if you already have openebs running and are adding/changing nodes
    - `kubectl edit node talos-storage-1` and change the value to the current node name
- Apply a patch to set some machine config stuff for OpenEBS which includes hugepages, a nodeLabel for where mayastor engine should run, and the `/var/local` bind mount:
```yaml
# ./patches/openebs.patch
machine:
  sysctls:
    vm.nr_hugepages: "1024"
  nodeLabels:
    openebs.io/engine: mayastor
  extraMounts:
    - destination: /var/local
      type: bind
      source: /var/local
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
  - Mount the disk to be passed to containers with appropriate privileges. This is required for openebs-io-engine to access the extra disk.
  ```yaml
  # ./patches/mount-sdb.patch
  machine:
    disks:
        - device: /dev/sdb
  ```
  - Apply: `talosctl patch mc -n 10.0.50.31 --patch @patches/mount-sdb.patch` - At this point the Talos node will reboot and should come back up healthy in a minute.
  - View the console or check the dashboard with `talosctl dashbord -n 10.0.50.31`
  - If you see an error about being unable to mount the disk or the partition being the wrong type, etc. you will need to wipe the disk and create a fresh GPT partition. As of Talos 1.9.0 this can be done with `talosctl wipe sdb -n 10.0.50.31`, otherwise you would need to do this outside of Talos.
    - `talosctl wipe disk sdb -n 10.0.50.31` - where `sdb` is the device, you confirmed this right? Confirm using `talosctl get disks -n 10.0.50.31`
    - Otherwise from Proxmox you could do this: Shut down the VM and do this in proxmox with `wipefs /dev/sdb` and then use `fdisk /dev/sdb` > `g` > `w` (`g` writes a new GPT table, `w` writes to disk). Now power Talos back on and it should do its thing.
    - Yet another option would be to boot into a different Linux ISO on the VM and use a tool like Gparted. Whatever you like best.

### Let's Verify Our Disk Mount
When Talos successfully mounts the extra disk, we should see it listed with `lsblk` without any partitions. We want to pass the raw disk to OpenEBS. In order to check, run a debug pod on your storage node and check the bind mounts.

- `kubectl debug node/talos-storage-1 -it --image=alpine -- /bin/sh`
- `apk add lsblk`
- `lsblk`
- Check for the mount, showing the full capacity of your disk.

Now repeat this whole process for any other Talos nodes you need. I have 3, so I'm doing `talos-storage-1`, `talos-storage-2` and `talos-storage-3`.

## Worker Nodes Also Need /var/local Mounted
Certain OpenEBS components run on any node, and this requires all worker nodes to have `/var/local` mounted.

```yaml
# ./patches/mount-var-local.patch
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

In my case, I applied this to my 3 worker nodes. I don't think a reboot is required, but you could if you wanted to:
- `talosctl patch mc -n 10.0.50.21 --patch @patches/mount-var-local.patch`
- `talosctl patch mc -n 10.0.50.22 --patch @patches/mount-var-local.patch`
- `talosctl patch mc -n 10.0.50.23 --patch @patches/mount-var-local.patch`

In order to check the other bind mount for `/var/local`, we have to wait until after deploying OpenEBS because the mount isn't utilized until a pod is deployed with a HostPath volume at or below this path. Specifically, the `openebs-io-engine-*` Daemonset maps to this path.

# Installing OpenEBS
This was a pain to figure out. Documentation from OpenEBS is lacking, and so is documentation from Talos on the same topic. Here's what I found to work. You need a privileged namespace, bind mounts on all worker nodes, then DiskPools before you can start testing PVCs.

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
- If pods are stuck initializing after a few minutes, start checking logs with `openebs-etcd-0` since a lot of things depend on that being up before it will initialize.
  - Related to `/var/local` bind mounts not being added to worker nodes, `openebs-etcd-*` will have Warning events about "FailedMount", stating that a PVC doesn't exist. If you look closely, the path starts with `/var/local/...` which means you're missing the `/var/local` bind mount on one or more Talos worker/storage nodes.
  - The fix in this situation would be to fix the bind mount on the worker nodes, and you could also try a reboot (you can identify the node based on the pod having issues). There's no need to change the OpenEBS deployment.
  - After fixing it, you might see stale pods with errors. You can terminate pods in an error state, and if they are needed the Deployment or Daemonset responsible for them will recreate them as needed.

## Add DiskPool(s)
I was excited at this point to test with a PVC, but then was confused about why it wouldn't provision anything. The Talos docs feel a bit sparse and seem to imply that now is the time to test with a PVC. But if you pay close attention they only mention testing the local provisioner, adding to my confusion. It turns out you need to add DiskPools, which makes sense in hindsight. If you've ever used Longhorn there is a similar config needed after the initial install, so that you know what disk capacity you have to work with.

- Earlier, we mounted that 2TB disk in the talos-storage-1 node. Now we'll use that for our first DiskPool
- Get the disk ID by exec-ing into the openebs-io-engine pod
  - Identify one of the io-engine pods: `kubectl get po -l app=io-engine -n openebs`
  - Exec into the pod: `kubectl exec -it openebs-io-engine-jpnrh -c io-engine -n openebs -- bin/sh`
  - `ls -lh /dev/disk/by-id/` - grab the one pointing to `/dev/sdb` in our case, which for me is `scsi-0QEMU_QEMU_HARDDISK_drive-scsi1`
- FYI I went with uring instead of aio since it's the new kid on the block
```yaml
# diskpool-1.yaml
apiVersion: "openebs.io/v1beta2"
kind: DiskPool
metadata:
  name: pool-1
  namespace: openebs
spec:
  node: talos-storage-1
  disks: ["uring:///dev/disk/by-id/scsi-0QEMU_QEMU_HARDDISK_drive-scsi1"]
```
- `kubectl apply -f diskpool-1.yaml`
- Verify: `kubectl get dsp -n openebs` - quickly it should change to Created state and Online POOL_STATUS

Repeat this process for any/all storage nodes you have. Since I'm virtualizing Talos, the disk path is exactly the same on all 3 nodes so I can reuse the config, just updating the pool name and the node name.

### Troubleshooting
Sorry I can't be a ton of help here since I have only done really limited troubleshooting. If you run into issues with diskpools stuck in Creating status, you will need to describe the dsp or check logs.

What I can say is that I'm running a homelab, which means I have 3 different disks, from 3 different manufacturers. Two of them worked in this configuration, but one did not. The disk is fine, it's fairly new, I've tried wiping it multiple times, multiple ways, but it just would not work with the diskpool. I tried doing it as a bind mount and using /mnt/local/nvme2tb (this one requires adding the volume to the io-engine Daemonset), mounting /dev/sdb, mounting /dev/sdb1, everything I could think of, but it would not create. I rebuilt the Talos node but got the same results. I switched to a different disk and changed nothing else, and it works totally fine. For prosperity, these are the disks I have and which ones worked in this configuration.
- Samsung 970 EVO Plus 2TB - no problems
- Samsung 990 EVO 2TB - no problems
- -WD BLACK SN770 2TB - no problems
- Crucial P3 Plus 2TB (CT2000P3PSSD8) - COULD NOT GET THIS WORKING :(

If you are reading this and you know or think you might know why this didn't work, please reach out! I'm interested in understanding why this wouldn't work and how I could troubleshoot better.

## Testing A Replicated PVC
If you are here, you have at least one working diskpool and are ready to test that PVC provisioning works and can be attached to a running pod. Let's test that.

- Verify diskpools: `kubectl get dsp -n openebs` - for my 3 storage nodes with 2TB volumes, I'm seeing this
```sh
NAME     NODE              STATE     POOL_STATUS   CAPACITY        USED   AVAILABLE
pool-1   talos-storage-1   Created   Online        1998443249664   0      1998443249664
pool-2   talos-storage-2   Created   Online        1998443249664   0      1998443249664
pool-3   talos-storage-3   Created   Online        1998443249664   0      1998443249664
```
- Check what storage classes are available: `kubectl get sc`
```sh
NAME                     PROVISIONER               RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
mayastor-etcd-localpv    openebs.io/local          Delete          WaitForFirstConsumer   false                  6h15m
mayastor-loki-localpv    openebs.io/local          Delete          WaitForFirstConsumer   false                  6h15m
openebs-hostpath         openebs.io/local          Delete          WaitForFirstConsumer   false                  6h15m
openebs-single-replica   io.openebs.csi-mayastor   Delete          Immediate              true                   6h15m
```
- For now, we're interested in testing that `openebs-single-replica` SC that uses Mayastor, so write this file. Note that this uses the default namespace:
```yaml
# test-pvc-openebs-single-replica.yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: openebs-testpvc
spec:
  storageClassName: openebs-single-replica
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```
- Apply it: `kubectl apply -f test-pvc-openebs-single-replica.yaml`
- Check PV and PVC:
  - `kubectl get pv`
  - `kubectl get pvc` - you should see a PVC named `openebs-testpvc` with status Bound and storage class `openebs-single-replica`
- Deploy a test pod to attach the PVC to - this pod is also in the default namespace:
```yaml
# pod-using-testpvc.yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: testlogger
spec:
    containers:
    - name: testlogger
      image: alpine:3.20
      command: ["/bin/ash"]
      args: ["-c", "while true; do echo \"$(date) - test log\" >> /mnt/test.log && sleep 1; done"]
      volumeMounts:
      - name: testvol
        mountPath: /mnt
    volumes:
    - name: testvol
      persistentVolumeClaim:
        claimName: openebs-testpvc
```
- `kubectl apply -f pod-using-testpvc.yaml`
- Verify it's running: `kubectl get po testlogger`
- Exec into the test pod: `kubectl exec -it testlogger -- /bin/sh`
- Look at your mounts with `df -h /mnt`. Since OpenEBS Mayastor uses NVMe-oF you should see that mount path `/mnt` attached to what looks like a NVMe block device.
```sh
Filesystem                Size      Used Available Use% Mounted on
/dev/nvme0n1              9.7G     28.0K      9.2G   0% /mnt
```
  - Hmm. The size doesn't quite add up. 9.7G is close to 10Gi but then 28.0K used does not equal 9.2G available. Hmm...
- Cleanup:
  - `kubectl delete -f pod-using-testpvc.yaml`
  - `kubectl delete -f test-pvc-openebs-single-replica.yaml`

## What Is A "Single" Replica Anyway?
> A replica is an exact reproduction of something...

A replica by definition cannot exist without copying something that already exists, which inherently means there must be at least 2. In the context of OpenEBS, a single replica just means you only have ONE copy of the data. It gets randomly assigned to one of the diskpools available. If that disk fails, that data is gone. But we are using OpenEBS for the purpose of replication, so how do we get more replicas??? Follow me.

- Create a new storage class with 2 replicas (feel free to do 3 or any value at this point):
```yaml
# openebs-2-replicas-sc.yaml
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: openebs-2-replicas
parameters:
  protocol: nvmf
  repl: "2"
provisioner: io.openebs.csi-mayastor
```
- `kubectl apply -f openebs-2-replicas-sc.yaml`
- Verify: `kubectl get sc`
- Test - deploy another test-pvc using the new SC:
```yaml
# test-pvc-openebs-2-replicas.yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: openebs-testpvc
spec:
  storageClassName: openebs-2-replicas
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```
- `kubectl apply -f openebs-2-replicas.yaml`
- Test with a pod, using the same pod from earlier - `kubectl apply -f pod-using-testpvc.yaml`
- Exec into the test pod: `kubectl exec -it testlogger -- /bin/sh`
- Check your mounted disk: `df -h /mnt`

# Conclusion
I'm going to stop here. I didn't go into any details about how to check which diskpools hold the replica(s) for the PV, but I assume there is a way to do that. I also did not look at how to recover in case there is a diskpool or storage node failure. Assuming you had 3 replicas, and one went down, there would be no data loss.

I didn't cover performance, monitoring, recovery, or anything else that you probably care about long term. That could be a future post, but my next stop is actually evaluating Longhorn with the v2 engine. As of today, they have released 1.8.0-rc5, which enables support for their v2 engine with Talos (this just means they support NVMe-oF). If Longhorn can now work with NVMe-oF and Talos, to me that is a much more mature and feature rich product, with more community support than OpenEBS. I believe it also supports snapshots and other features that OpenEBS does not currently.

My next post will be all about blowing this setup away and doing it all over with Longhorn. Hopefully by then the stable 1.8.0 will have been released.