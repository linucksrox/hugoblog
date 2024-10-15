---
layout: blog
draft: false
title: Kubernetes Homelab Series Part 1 - Introduction and Talos Installation (WIP)
date: 2024-08-27T23:06:24.328Z
tags:
  - kubernetes
  - homelab
  - talos
summary: This in depth series will walk through building a Kubernetes cluster beyond the basics, including dynamically provisioned storage, certificate
  management, and backups.
---
# Introduction

This series will go through the process of building a Kubernetes cluster on Proxmox using Talos Linux from the ground up. I intend for this to go into a lot more depth than a lot of basic tutorials, and by the end you should have a cluster that you can actually trust to run workloads with high availability, backups, etc.

I use Proxmox in my homelab, but this could be followed easily in ESXi or other hypervisors of choice.

Building a bare cluster is easy. Making it work for you is not. Not because it's hard, but because there are a lot of components and decisions to make. As many problems as Kubernetes solves, it introduces many new challenges and new ways of thinking about things. One of the hardest problems in my opinion is storage, which we will talk about and I will show you how I decided to do this in my homelab. It is possible to achieve dynamically provisioned storage with a single disk, and not just with hostPath. You don't have to use Rancher Longhorn, OpenEBS (Mayastor) or Portworx. You don't have to use NFS. It's mix and match!

Other topics will include LoadBalancer services using MetalLB, secret management using Sealed Secrets (and also talking about SOPS with age), cluster backup of the etcd cluster along with Velero to back up cluster resources, and ingress with Traefik (plus Gateway API which is the successor to ingress controllers).

Finally, and this will actually be the first topic, I will be using Talos Linux because I believe it's the easiest option overall considering the initial build and also ongoing maintenance such as OS upgrades and Kubernetes version upgrades.

Using Talos Linux in theory makes managing the cluster itself very simple because their API is the only way to make changes and they include safe upgrade options for both the Talos nodes and Kubernetes versions. It does introduce some complexity, especially relating to storage, but we'll talk about that a little later.

# Prerequisites and Assumptions

- You have a homelab of some sort and can download ISOs and create VMs.
- You have a disk or virtual disk to dedicate for storage (or you can do clustered storage, we'll talk about it).
- You are able to read and follow along :)

# Getting Started With Talos (Overview)

https://www.talos.dev/  
Talos Linux is Linux designed for Kubernetes – secure, immutable, and minimal.

- Supports cloud platforms, bare metal, and virtualization platforms
- All system management is done via an API. No SSH, shell or console
- Production ready: supports some of the largest Kubernetes clusters in the world
- Open source project from the team at Sidero Labs

It's the easiest solution to deploy (and maintain) a Kubernetes cluster.

## Proxmox Setup
- Download latest Talos Linux release, grabbing metal-amd64.iso version from the GitHub releases page https://github.com/siderolabs/talos/releases
- Upload it to proxmox ISOs, or have proxmox download it directly from the URL
- Optionally rename the file manually on the filesystem: /tank/isos/template/iso/
- Create VMs in Proxmox (3 control plane nodes, 3 worker nodes)
- For OS, the talos metal-amd64 iso image. It will be replaced later during bootstrapping, so getting the latest image with added extensions now is not important.
  - Enable QEMU guest agent (requires qemu guest extensions in Talos)
  - For disks, choose something on SSD storage (required for reasonable etcd performance)
  - Enable Discard and SSD emulation options
  - 20GB should be plenty
  - 4CPU (2 minimum)
  - 4GB RAM (2 minimum)
  - VLAN 50, DHCP (VLAN is optional, use whatever VLAN you want for your cluster if you have one, otherwise leave this blank)
- Start the VMs, and check the console for each to verify they are running (and have an IP assigned on the kubernetes VLAN if applicable)

## Talos Linux Setup
- Check out the talos project: https://www.talos.dev/v1.8/talos-guides/install/virtualized-platforms/proxmox/
- On whichever machine you are using to manage Talos such as your local workstation, install talosctl and verify it's the same version you are deploying
  - See https://www.talos.dev/v1.8/talos-guides/install/talosctl/
  - The recommended method is `brew install siderolabs/tap/talosctl`
  - The alternative method is `curl -sL https://talos.dev/install | sh`
  - Note: if you are upgrading talosctl, the `curl` method will tell you it's already installed with a message saying `To force re-downloading, delete '/usr/local/bin/talosctl' then run me again.`
    - Delete the current binary with `sudo rm -rf /usr/local/bin/talosctl`
    - Re-run `curl -sL https://talos.dev/install | sh`
- Create a new folder for your Talos cluster. You will be initializing a git repo here and eventually pushing to a remote repo.
  - `mkdir -p taloscluster && cd taloscluster`
- Generate new Talos secrets. These are used to authenticate with the Talos cluster.
  - `talosctl gen secrets`
  - This stores a file in the current directory named `secrets.yaml`
- Build the Talos image you will be using to install Talos to each node. This allows you to add extensions which add capabilities such as iSCSI support and QEMU guest agent for Proxmox.
  - Go to https://factory.talos.dev/ - Bare Metal Machine > 1.8.1 (or the version you want) > AMD64 > Check siderolabs/qemu-guest-agent > Check siderolabs/iscsi-tools > (no customization) > copy the factory image string under "Initial Install" section, e.g. `factory.talos.dev/installer/88d1f7a5c4f1d3aba7df787c448c1d3d008ed29cfb34af53fa0df4336a56040b:v1.8.1`
- Generate base machine configs: https://www.talos.dev/v1.8/talos-guides/install/virtualized-platforms/proxmox/#generate-machine-configurations
  - The `CONTROL_PLANE_IP` should be the IP of one of the control plane nodes, or optionally the virtual IP if you plan on load balancing the control plane (highly recommended for HA: https://www.talos.dev/v1.8/talos-guides/network/vip/)
  - Feel free to change the cluster name (default is "talos-proxmox-cluster")
    ```bash
    talosctl gen config talos-proxmox-cluster https://$CONTROL_PLANE_IP:6443 --with-secrets secrets.yaml --output-dir _out --install-image factory.talos.dev/installer/88d1f7a5c4f1d3aba7df787c448c1d3d008ed29cfb34af53fa0df4336a56040b:v1.8.1
    ```
- In Part 2, we will discuss encrypting YAML files with SOPS + age, which you can use to encrypt this file and store it securely in a git repo. The TLDR steps for now are as follows:
  - Install `sops` and `age` binaries locally
  - Generate a key pair with `age-keygen`
  - Create a local SOPS config file named `.sops.yaml` with the following contents. This tells SOPS which files to encrypt, and which encryption key to use.
  ```yaml
  ---
  creation_rules:
    - path_regex: /*secrets(\.encrypted)?.yaml$
      age: replace-with-your-public-key
    - path_regex: /*talosconfig(\.encrypted)?$
      age: replace-with-your-public-key
    - path_regex: /*controlplane(\.encrypted)?.yaml$
      encrypted_regex: "(^token|crt|key|id|secret|secretboxEncryptionSecret)$"
      age: age1gkhxw26n8nknr6f7tsd3c56su775t5kxqs5w45k9xjz2htqnh5fsmppv6l
    - path_regex: /*worker(\.encrypted)?.yaml$
      encrypted_regex: "(^token|crt|key|id|secret|secretboxEncryptionSecret)$"
      age: age1gkhxw26n8nknr6f7tsd3c56su775t5kxqs5w45k9xjz2htqnh5fsmppv6l
  ```
  - Encrypt secrets.yaml: `sops --encrypt secrets.yaml > secrets.encrypted.yaml`
  - Encrypt talosconfig: `sops --encrypt _out/talosconfig > talosconfig.encrypted`
  - Encrypt controlplane.yaml: `sops --encrypt _out/controlplane.yaml > _out/controlplane.encrypted.yaml`
  - Encrypt worker.yaml: `sops --encrypt _out/worker.yaml > _out/worker.encrypted.yaml`
- **IMPORTANT!** Now's a good time to create `.gitignore` and make sure you exclude `secrets.yaml` and `talosconfig` from your git repo
```ini
secrets.yaml
talosconfig
controlplane.yaml
worker.yaml
```
- Initialize the git repo: `git init`
- Make your initial git commit: `git add .`
- Double check that only encrypted yaml files and talosconfig are staged: `git status`
- Commit: `git commit -m "initial commit with encrypted cluster secrets and talosconfig"`
- Create node specific patches (used for defining a static IP, etc.): https://www.talos.dev/v1.8/talos-guides/configuration/patching/
  - You can install with the base config (controlplane.yaml or worker.yaml), then apply node specific patches afterward. You could also duplicate the whole thing for each node with specific customizations such as static IP, but that can complicate upgrades and make it harder to keep things encrypted.
  - `mkdir -p patches`
  - Create patches for each controlplane and worker node:
    - cp1.yaml
    ```yaml
    machine:
      network:
        hostname: taloscp1
        interfaces:
          - deviceSelector:
              busPath: "0*" # This is an example; adjust based on your hardware
            addresses:
              - 10.0.50.161/24
            routes:
              - network: 0.0.0.0/0
                gateway: 10.0.50.1
            vip:
              ip: 10.0.50.160
        nameservers:
          - 192.168.1.22
    ```
    - cp2.yaml
    ```yaml
    machine:
      network:
        hostname: taloscp2
        interfaces:
          - deviceSelector:
              busPath: "0*" # This is an example; adjust based on your hardware
            addresses:
              - 10.0.50.162/24
            routes:
              - network: 0.0.0.0/0
                gateway: 10.0.50.1
            vip:
              ip: 10.0.50.160
        nameservers:
          - 192.168.1.22
    ```
    - cp3.yaml
    ```yaml
    machine:
      network:
        hostname: taloscp3
        interfaces:
          - deviceSelector:
              busPath: "0*" # This is an example; adjust based on your hardware
            addresses:
              - 10.0.50.163/24
            routes:
              - network: 0.0.0.0/0
                gateway: 10.0.50.1
            vip:
              ip: 10.0.50.160
        nameservers:
          - 192.168.1.22
    ```
    - wk1.yaml
    ```yaml
    machine:
      network:
        hostname: taloswk1
        interfaces:
          - deviceSelector:
              busPath: "0*" # This is an example; adjust based on your hardware
            addresses:
              - 10.0.50.171/24
            routes:
              - network: 0.0.0.0/0
                gateway: 10.0.50.1
        nameservers:
          - 192.168.1.22
    ```
    - wk2.yaml
    ```yaml
    machine:
      network:
        hostname: taloswk2
        interfaces:
          - deviceSelector:
              busPath: "0*" # This is an example; adjust based on your hardware
            addresses:
              - 10.0.50.172/24
            routes:
              - network: 0.0.0.0/0
                gateway: 10.0.50.1
        nameservers:
          - 192.168.1.22
    ```
    - wk3.yaml
    ```yaml
    machine:
      network:
        hostname: taloswk3
        interfaces:
          - deviceSelector:
              busPath: "0*" # This is an example; adjust based on your hardware
            addresses:
              - 10.0.50.173/24
            routes:
              - network: 0.0.0.0/0
                gateway: 10.0.50.1
        nameservers:
          - 192.168.1.22
    ```
- Add your patches to the git repo: `git add .` and `git commit -m "add patches"`
- Push your git repo somewhere like GitHub or GitLab (I'll just assume you know how already or can search DuckDuckGo)

## Talos Linux Installation
Finally, time to actually install Talos!

- Check the console for Talos CP1 in Proxmox to get the IP address (should have been assigned via DHCP). Let's say it's 10.0.50.129
- Install using base controlplane.yaml config: `talosctl apply-config --insecure --nodes 10.0.50.129 --file _out/controlplane.yaml`
- Follow this same process for each of the other 2 controlplane nodes.
- Follow the same process for all worker nodes, but use `_out/worker.yaml`: `talosctl apply-config --insecure --nodes 10.0.50.132 --file _out/worker.yaml`
- Watch the console in Proxmox to see it install and reboot. When you see the Kubernetes version and Kubelet status Healthy on all 6 nodes, you can proceed to patching each node to assign static IPs.
- Patch each node using the corresponding patch file. Endpoint should be the control plane node you're targeting, for controlplane setup. For workers, the endpoint needs to be one of the control plane nodes, and since you've already updated to 10.0.50.161 in this example, you can use the first control plane as the endpoint when patching all the worker nodes:
  -  `talosctl patch mc -e 10.0.50.129 -n 10.0.50.129 --patch @patches/cp1.yaml`
  -  `talosctl patch mc -e 10.0.50.130 -n 10.0.50.130 --patch @patches/cp2.yaml`
  -  `talosctl patch mc -e 10.0.50.131 -n 10.0.50.131 --patch @patches/cp3.yaml`
  -  `talosctl patch mc -e 10.0.50.161 -n 10.0.50.132 --patch @patches/wk1.yaml`
  -  `talosctl patch mc -e 10.0.50.161 -n 10.0.50.133 --patch @patches/wk2.yaml`
  -  `talosctl patch mc -e 10.0.50.161 -n 10.0.50.134 --patch @patches/wk3.yaml`
- Note: the VIP doesn't come online until after bootstrapping the cluster. Don't be like me and try to troubleshoot this right now :)
- Also, apply this other patch to all nodes for automatic kubelet cert rotation which is required for metrics-server (if you want to use that). Save in `./patches/kubelet-cert-rotation.patch`
  ```yaml
  ---
  machine:
    kubelet:
      extraArgs:
        rotate-server-certificates: true
  
  cluster:
    extraManifests:
      - https://raw.githubusercontent.com/alex1989hu/kubelet-serving-cert-approver/main/deploy/standalone-install.yaml
  ```
  -  `talosctl patch mc -e 10.0.50.161 -n 10.0.50.161 --patch @patches/kubelet-cert-rotation.patch`
  -  `talosctl patch mc -e 10.0.50.161 -n 10.0.50.162 --patch @patches/kubelet-cert-rotation.patch`
  -  `talosctl patch mc -e 10.0.50.161 -n 10.0.50.163 --patch @patches/kubelet-cert-rotation.patch`
  -  `talosctl patch mc -e 10.0.50.161 -n 10.0.50.171 --patch @patches/kubelet-cert-rotation.patch`
  -  `talosctl patch mc -e 10.0.50.161 -n 10.0.50.172 --patch @patches/kubelet-cert-rotation.patch`
  -  `talosctl patch mc -e 10.0.50.161 -n 10.0.50.173 --patch @patches/kubelet-cert-rotation.patch` 
- Configure talosctl by either copying talosconfig to $HOME/.talos/config or exporting the ENV variable:
  - `cp _out/talosconfig ~/.talos/config`
  - OR
  - `export TALOSCONFIG="_out/talosconfig"`
- Set endpoints list and nodes list if youi get tired of specifying them every time you run talosctl commands:
  - `talosctl config endpoint 10.0.50.161 10.0.50.162 10.0.50.163`
  - `talosctl config node 10.0.50.161 10.0.50.162 10.0.50.163 10.0.50.171 10.0.50.172 10.0.50.173`
- Bootstrap the cluster:
  - `talosctl bootstrap -n 10.0.50.161`
- Grab kubeconfig - download to current directory
  - `talosctl kubeconfig -n 10.0.50.161 .`
- Move kubeconfig if this will be used as your default cluster
  - `mv kubeconfig ~/.kube/config`
- Install kubectl
  - `curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"`
  - `install -m 755 /usr/local/bin/kubectl`
  - `kubectl version`
- TEST!
  - `kubectl get node -o wide`
  ```
  NAME       STATUS   ROLES           AGE     VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE         KERNEL-VERSION   CONTAINER-RUNTIME
  taloscp1   Ready    control-plane   6m34s   v1.31.1   10.0.50.161   <none>        Talos (v1.8.1)   6.6.54-talos     containerd://2.0.0-rc.5
  taloscp2   Ready    control-plane   6m34s   v1.31.1   10.0.50.162   <none>        Talos (v1.8.1)   6.6.54-talos     containerd://2.0.0-rc.5
  taloscp3   Ready    control-plane   6m2s    v1.31.1   10.0.50.163   <none>        Talos (v1.8.1)   6.6.54-talos     containerd://2.0.0-rc.5
  taloswk1   Ready    <none>          6m26s   v1.31.1   10.0.50.171   <none>        Talos (v1.8.1)   6.6.54-talos     containerd://2.0.0-rc.5
  taloswk2   Ready    <none>          6m22s   v1.31.1   10.0.50.172   <none>        Talos (v1.8.1)   6.6.54-talos     containerd://2.0.0-rc.5
  taloswk3   Ready    <none>          6m18s   v1.31.1   10.0.50.173   <none>        Talos (v1.8.1)   6.6.54-talos     containerd://2.0.0-rc.5
  ```
- In testing I noticed some stuff that's supposed to run on all control plane nodes was only runninn on cp1 and cp3, so check that and reboot each control plane node one at a time to resolve if you see the same issue. You would be looking for `kube-apiserver-[node]`, `kube-controller-manager-[node]`, and `kube-scheduler-[node]`.
  - `kubectl get po -n kube-system`
  - `talosctl reboot -n 10.0.50.161`
  - `talosctl reboot -n 10.0.50.162`
  - `talosctl reboot -n 10.0.50.163`

# Next Steps
Now you should have a Talos Linux cluster running with each node having its own static IP, along with a VIP for the control plane cluster. You are ready to start installing what I would consider foundational components that will be used to automate tasks for you when deploying actual workloads later on. These include:

- Encrypted secret manager - sealed secrets
- LoadBalancer service - MetalLB
- Certificate manager - cert-manager
- CSI driver - democratic-csi
- Ingress provider - traefik
- Backup manager - velero
- Metrics server
