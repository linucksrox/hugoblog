---
layout: blog
draft: false
title: Kubernetes Homelab Series Part 7 - Backups With Velero (WIP)
date: 2024-11-30
tags:
  - kubernetes
  - homelab
  - backup
  - etcd
summary: A look into backing up etcd in a Talos Linux cluster, along with full cluster resource backups using Velero
---
# What Do You Back Up In Kubernetes?
It's easy to gloss over backups especially when you are setting up a new cluster, finally getting stuff running on it, and feeling excited that you can now deploy some service and basically automate certificates, ingress, and everything else that used to be a pain to do manually. But as soon as something fails, you will wish you spent a little more time not only thinking about backups, but also documenting step by step how to recover from a disaster.

In Kubernetes, we have two main things to back up: persistent data, and the state of the cluster resources.

With Velero, backups in Kubernetes are EASY. Just do it now, before you put stuff on the cluster, and write down the steps to recover. Why not?

# etcd Backups In Talos Linux
Before we dive into Velero, I want to mention that theoretically, you could just use Velero for backups and never worry about etcd. However, it's a best practice to backup etcd and it's easy to do, there's no reason not to have another recovery point.

What is etcd (I think it's pronounced "et-see-dee")? This is the database that the control plane nodes use to keep track of any and all resources in the cluster. It's the cluster database that keeps track of every resource, status, etc. in the cluster. This includes every node, pod, deployment, secret, etc. If you had a catastrophe and lost all your cluster resource state, or had something get corrupted, you should be able to restore the etcd database and get back to the exact state it was in when the backup was taken.

Side note: This tracks real time data so whatever storage the etcd cluster is running on (the actual disks the control plane node stores this data on) needs to be pretty fast. If it's slow, you can run into issues where `kubectl` commands are very slow, or stuff stops working as expected in the cluster. Maybe this note belongs in the Talos Linux setup section, but here we are.

Talos Linux, unlike a traditional kubeadm install, does not run etcd as a static pod. Instead, it's hidden behind the Talos API, meaning you have to use `talosctl` to manage it: https://www.talos.dev/v1.8/advanced/etcd-maintenance/

## How To Back Up etcd In Talos
https://www.talos.dev/v1.8/advanced/disaster-recovery/#backup

- `talosctl -n <IP> etcd snapshot /backup_path/etcd.snapshot.$(/bin/date +%Y%m%d)`
  - You need to pass any one of the control plane node IPs to the `-n` flag

You could automate this somewhere so you have regular backups. Highly recommended! If you do a bash script, you'll need to pass the `--talosconfig /path/to/talosconfig` option.

If you're currently in a Disaster Recovery scenario and the snapshot command is failing, copy the etcd database directly: https://www.talos.dev/v1.8/advanced/disaster-recovery/#disaster-database-snapshot
- `talosctl -n <IP> cp /var/lib/etcd/member/snap/db .`

## How To Recover etcd In Talos
https://www.talos.dev/v1.8/advanced/disaster-recovery/#recovery

I haven't done this process yet (and hopefully I don't need it). Follow the official guide.

# What's Velero?
https://velero.io/

Velero is an open source DR and backup/migration/recovery tool for Kubernetes. It backs up all cluster resources (or whatever you specify) and can be used for backup, restore, disaster recovery and migrating to new clusters. It's also capable of backing up PVs. Backups are stored on external endpoints (mostly S3 capable services including MinIO). Backup/restore is managed through CRDs. There are two components: cluster resources and a client side utility `velero`.

Don't trust their blog to be up to date because the current release according to the blog is 1.11, released April 26, 2023, although I'm currently running 1.15 which was released November 5, 2024.

Get an accurate changelog right from the source: https://github.com/vmware-tanzu/velero/releases

# Velero Installation/Setup
I'll walk through installing Velero with some options you might want to consider. Then I'll dive into running a backup and restore manually, scheduling backups, and backing up persistent volumes with Kopia.

## Prerequisites
- Access to a Kubernetes cluster, v1.16 or later, with DNS and container networking enabled. For more information on supported Kubernetes versions, see the Velero [compatibility matrix](https://github.com/vmware-tanzu/velero#velero-compatibility-matrix).
- `kubectl` installed locally
- Object storage. I'm running MinIO in my lab. You could use MinIO, Ceph, AWS S3, Backblaze B2 (S3 API), or any other object storage you prefer.

## Installing Velero - Client Side
https://velero.io/docs/v1.15/basic-install/

- Install the `velero` client utility
  ```bash
  VERSION=v1.15.0
  curl -LJO https://github.com/vmware-tanzu/velero/releases/download/$VERSION/velero-$VERSION-linux-amd64.tar.gz
  tar zxvf velero-$VERSION-linux-amd64.tar.gz
  install -m 755 velero-$VERSION-linux-amd64/velero /usr/local/bin/velero
  rm -rf velero-$VERSION-linux-amd64
  rm -rf velero-$VERSION-linux-amd64.tar.gz
  velero version
  ```

## Installing Velero - Server Side
There are two installation methods, using the Helm chart or using the `velero` utility. Let's use `velero` CLI since the Helm method seems complicated based on what I can find in the docs.

### MinIO Setup
We need to start with an access key for the S3 backup target. I'm using MinIO: https://velero.io/docs/v1.15/contributions/minio/
- Create a bucket `velero-talos`
- Create a group `velero-svc`
- Create a service account user `velero` and add to `velero-svc` group
- Create a policy `velero-rw`. Raw Policy:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket",
                "s3:PutObject",
                "s3:DeleteObject",
                "s3:GetObject"
            ],
            "Resource": [
                "arn:aws:s3:::velero-talos",
                "arn:aws:s3:::velero-talos/*"
            ]
        }
    ]
}
```
- Assign that policy to the `velero-svc` group
- Create an access key on the `velero` user
- Create a file named `minio-access-key.txt`, replacing the values from your access key
```ini
[default]
aws_access_key_id = minio
aws_secret_access_key = minio123
```
  - Decide what you think is best for storing this file or the credentials somewhere  securely. I would suggest using SOPS, but I will leave the implementation up to you.

### Install Using `velero` Utility
- Arguments reference:
  - `--provider` - Use the AWS provider which is also used for any generic S3 object storage, including MinIO
  - `--secret-file` - Specifies the secrets to use for authentication against the MinIO storage
  - `--plugins` - Required for Velero to connect to an S3 backend
  - `--bucket` - specifies the name of the bucket on the S3 backend
  - `--use-volume-snapshots=true` - Enables volume snapshots
  - `--backup-location-config` - Configures the connection URL and options for the S3 backend
  - `--features=EnableCSI` - Enables Velero to use a built in CSI snapshot driver, such as democratic-csi and take snapshots using a volumesnapshotclass. To be clear, Velero would create a volumesnapshot and it would be stored wherever democratic-csi stores snapshots (in our case that's in another dataset on the TrueNAS instance). This is the easy button if you already have CSI snapshots enabled, but the alternative is using Kopia and storing snapshots on the MinIO backend.
- Install:
  - `velero install --provider aws --secret-file minio-access-key.txt --plugins velero/velero-plugin-for-aws:v1.11.0 --bucket velero --use-volume-snapshots=true --backup-location-config region=us-east-1,s3ForcePathStyle="true",s3Url=http://192.168.1.35:9000 --features=EnableCSI`
    - You may need to update the plugin version and s3URL.
    - Optionally, set `use-volume-snapshots=false` if you don't want to back up PVs.
- Verify:
  - `kubectl get all -n velero`
- Post install:
  - If you are using democratic-csi for snapshots, you also need to add a label on the VolumeSnapshotClass to let Velero know which one to use by default. This must only be set on 1 volumeSnapshotClass
    - `kubectl edit volumesnapshotclass truenas-iscsi`
    ```yaml
      velero.io/csi-volumesnapshot-class: "true"
    ```
  - If you're ADDING this funtionality after you've already installed Velero, you just need to modify the server deployment and also enable on the client: velero.io/csi-volumesnapshot-class: "true"
    - Server:
      - `kubectl edit -n velero deploy/velero`
      ```yaml
      --features=EnableCSI
      ```
    - Client:
      - `velero client config set features=EnableCSI`
      - `velero client config get features`

## Testing
### Manual Backup/Restore
Let's back up the traefik namespace:
- `velero backup create traefik-backup --include-namespaces traefik`
- Verify: `velero backup describe traefik-backup`
  - Looking for Phase: Complete
- Test a DR scenario:
  - Blow away the traefik namespace: `kubectl delete ns traefik`
  - Restore traefik-backup with Velero: `velero restore create --from-backup traefik-backup`
  - Verify: `kubectl get ns` or `velero restore describe traefik-backup-20241130181633` (get the restore name from the previous restore command)

## Create A Backup Schedule


## Back Up PVCs Using democratic-csi Snapshots
If you enabled snapshots in democratic-csi and also enabled snapshos in Velero as described above, then anything you back up with Velero that includes a PVC will be snapshotted. I tested this by deploying a pod/deployment in the default namespace, attached to a test PVC, then ran a Velero backup against it.

- `velero backup create test-manual-backup` (this backs up everything in the cluster)

## Kopia