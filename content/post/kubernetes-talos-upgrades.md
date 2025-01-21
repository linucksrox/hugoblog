---
layout: blog
draft: false
title: Upgrade Talos Linux and Kubernetes (WIP)
date: 2025-01-21
tags:
  - kubernetes
  - homelab
  - talos
  - upgrade
summary: A quick rundown of Talos OS upgrades and Kubernetes version upgrades
---
# Upgrade Kubernetes Version
https://www.talos.dev/v1.9/kubernetes-guides/upgrading-kubernetes/

> Upgrading Kubernetes is non-disruptive to the cluster workloads.

You can do this live, assuming you don't have single-replica workloads that are node-specific.

Today I will be upgrading to Kubernetes version `v1.31.5`. I'm currently on `v1.30.0` but I want to make sure I'm running the same version that is being tested on the CKA exam that I'm studying for which is currently 1.31.

Talos recommends using the `talosctl upgrade-k8s` command which automatically upgrades the entire cluster and has built in safety checks. They explain how to do it manually, but I chose Talos Linux partly based on the ease of ongoing maintenance and upgrades so I will be using the easy button here!

- Check current version: `kubectl get node`
- Upgrade: `talosctl -n 10.0.50.11 upgrade-k8s --to 1.31.5`
  - You only need to specify a single control plane node, but this will upgrade the whole cluster
  - You need to choose a real Kubernetes version, and omit the `v` from this command - https://kubernetes.io/releases/
  - This will take a while, so try to be patient.
- Verify version: `kubectl get node`
- That's it. You are now done.

# Upgrade Talos OS
https://www.talos.dev/v1.9/talos-guides/upgrading-talos/

The Talos team recommends using the same version of `talosctl` that your nodes are running. You will then upgrade `talosctl` after the node upgrades are complete.

Be sure to upgrade one node at a time and check that it's healthy before moving on. You can blast through them using a for loop, or do them by hand. Just don't do them all at the same time :)

- Check versions:
  - Client: `talosctl version --client`
  - Server: `kubectl get node -o wide` (OS-IMAGE column)
- Get a new Talos OS image from the factory: https://factory.talos.dev
  - Make sure to add any existing extensions you're using such is `iscsi-tools`
  - Copy the image string under the "Upgrading Talos Image" header. In my case this looks like `factory.talos.dev/installer/88d1f7a5c4f1d3aba7df787c448c1d3d008ed29cfb34af53fa0df4336a56040b:v1.9.2`
- Upgrade one node: `talosctl upgrade -n 10.0.50.11 --image factory.talos.dev/installer/88d1f7a5c4f1d3aba7df787c448c1d3d008ed29cfb34af53fa0df4336a56040b:v1.9.2 --preserve`
  - `-n`: Specify the node to upgrade
  - `--image`: Specify the factory image to use
  - `--preserve`: Don't wipe extraMounts if applicable. I default to using this unless I have a specific reason to wipe additional mounts.
- Repeat the upgrade command for each node, one at a time, until all nodes have been upgraded.

In my homelab, I am comfortable blasting through upgrades with a for loop, so my upgrade command looks like this:
```bash
for node in 11 12 13 21 22 23 31 32 33; do talosctl upgrade -n 10.0.50.11 --image factory.talos.dev/installer/88d1f7a5c4f1d3aba7df787c448c1d3d008ed29cfb34af53fa0df4336a56040b:v1.9.2 --preserve; done
```

Once Nodes have been upgraded, upgrade the `talosctl` client so the version matches the Talos node version.
- `rm /usr/local/bin/talosctl`
- `curl -sL https://talos.dev/install | sh` (this gets the latest version)
  - You can also download a specific release from https://github.com/siderolabs/talos/releases, e.g. `curl -LJO https://github.com/siderolabs/talos/releases/download/v1.8.3/talosctl-linux-amd64`
- Verify: `talosctl version --client`

That's it!