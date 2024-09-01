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

We are going to cover building a Kubernetes cluster from the ground up, suitable for a homelab or production use (mostly). I will go into much greater depth than a lot of tutorials do, well beyond "the basics."

I use Proxmox in my homelab, but this could be followed easily in VMWare or other hypervisors of choice.

Building a bare cluster is easy. Making it work for you is not. Not because it's hard, but there are a lot of pieces and decisions to make. As many problems as Kubernetes solves, it introduces many new challenges and ways of thinking about things. One of the hardest problems is storage, which we will talk about and I will demonstrate how to do this in a homelab. Storage can be done with a single disk, and not just for hostPath. You don't have to use Rancher Longhorn. And you don't even have to use NFS. We'll be looking at NFS and NVMe-oF and how to actually get them working (for reference: https://ssdcentral.net/getting-started-with-nvme-over-fabrics-with-tcp/).

Other topics will include LoadBalancer services using MetalLB, secret management using Sealed Secrets (and also talking about SOPS with age), cluster backup of the etcd cluster along with Velero to back up cluster resources, and ingress with Traefik (plus Gateway API which is the successor to ingress controllers).

Finally, and this will actually be the first topic, I will be using Talos Linux because I believe it's the easiest option overall considering the initial build and also ongoing maintenance such as OS upgrades and Kubernetes version upgrades.

# Prerequisites and Assumptions

- You have a homelab of some sort and can download ISOs and create VMs.
- You have a disk or virtual disk to dedicate for storage (or you can do clustered storage, we'll talk about it)
- ... I think that's it? For now at least.

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
  - VLAN 50, DHCP (optional)
- Start the VMs, and check the console for each to verify they are running (and have an IP assigned on the kubernetes VLAN if applicable)

## Talos Linux Setup
- Check out the talos project: https://www.talos.dev/v1.7/talos-guides/install/virtualized-platforms/proxmox/
- On whichever machine you are using to manage Talos such as your local workstation, install talosctl and verify it's the same version you are deploying
  - See https://www.talos.dev/v1.7/talos-guides/install/talosctl/
  - The recommended method is `brew install siderolabs/tap/talosctl`
  - The alternative method is `curl -sL https://talos.dev/install | sh`
  - Note: if you are upgrading talosctl, the `curl` method will tell you it's already installed with a message saying `To force re-downloading, delete '/usr/local/bin/talosctl' then run me again.`
    - Delete the current binary with `sudo rm -rf /usr/local/bin/talosctl`
    - Re-run `curl -sL https://talos.dev/install | sh`
- Generate new Talos secrets. These are used to authenticate with the Talos cluster.
  - `talosctl gen secrets`
  - This stores a file in the current directory named `secrets.yaml`
  - **IMPORTANT!** Make a backup copy of `secrets.yaml` and store it securely.
  - In Part 2, we will discuss encrypting YAML files with SOPS + age, which you can use to encrypt this file and store it securely in a git repo. The TLDR steps for now are as follows:
    - Install `sops` and `age` binaries locally
    - Generate a key pair with `age-keygen`
    - Create a local SOPS config file named `.sops.yaml` with the following contents. This tells SOPS which files to encrypt, and which encryption key to use.
      ```
      ---
      creation_rules:
        - path_regex: /*secrets(\.encrypted)?.yaml$
          age: replace-with-your-public-key
        - path_regex: /*talosconfig(\.encrypted)?$
          age: replace-with-your-public-key
      ```
    - Encrypt the file `sops --encrypt secrets.yaml > secrets.encrypted.yaml`
    - **IMPORTANT!** Only store the encrypted version of the file in the git repo, which means the best practice would be to add `secrets.yaml` to .gitignore so it doesn't get committed. You would clone the repo, decrypt `secrets.encrypted.yaml` to `secrets.yaml` and use that temporarily to build a new node, etc. if needed.
- Enable QEMU support: https://www.talos.dev/v1.7/talos-guides/install/virtualized-platforms/proxmox/#qemu-guest-agent-support-iso
  - Go to https://factory.talos.dev/ - Bare Metal Machine > 1.7.6 (or the version you want) > AMD64 > Check siderolabs/qemu-guest-agent > (no customization) > copy the factory image string under "Initial Install" section, e.g. `factory.talos.dev/installer/376567988ad370138ad8b2698212367b8edcb69b5fd68c80be1f2ec7d603b4ba:v1.7.6`
- Generate base machine configs: https://www.talos.dev/v1.7/talos-guides/install/virtualized-platforms/proxmox/#generate-machine-configurations
  - The CONTROL_PLANE_IP should be the IP of one of the control plane nodes, or optionally the virtual IP if you plan on load balancing the control plane (highly recommended for HA: https://www.talos.dev/v1.7/talos-guides/network/vip/)
  - Feel free to change the cluster name (default is "talos-proxmox-cluster")
  - `talosctl gen config talos-proxmox-cluster https://$CONTROL_PLANE_IP:6443 --output-dir _out --install-image factory.talos.dev/installer/376567988ad370138ad8b2698212367b8edcb69b5fd68c80be1f2ec7d603b4ba:v1.7.6`
  - **IMPORTANT!** Make a backup copy of `talosconfig` and store it securely. See above about encrypting this file with SOPS and only storing the encrypted version in a git repo.
- Create patches (elaboration TBD)
- Store talos secret somewhere (git repo)
- Apply foundational components in order
- Encrypted secret manager - sealed secrets - TBD
- LoadBalancer service - TBD
- Certificate manager - cert-manager - TBD
- CSI driver - democratic-csi - TBD
- Ingress provider - traefik - TBD
- Backup manager - velero - TBD
