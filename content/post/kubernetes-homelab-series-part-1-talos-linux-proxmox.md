---
layout: blog
draft: false
title: Kubernetes Homelab Series Part 1 - Introduction and Talos Installation
date: 2024-10-10
tags:
  - kubernetes
  - homelab
  - talos
summary: This in depth series will walk through building a Kubernetes cluster beyond the basics, including dynamically provisioned storage, certificate management, and backups.
---
# Introduction

This series will go through the process of building a Kubernetes cluster on Proxmox using Talos Linux from the ground up. I intend for this to go into a lot more depth than a lot of basic tutorials, and by the end you should have a cluster that you can actually trust to run workloads with high availability, backups, etc.

The goal of this series is not to be a Kubernetes tutorial, but more of a step by step to building a Talos Linux cluster that has all the components you might need, along with some insights I've learned over time that you might find helpful.

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
  - The `CONTROL_PLANE_IP` needs to point to one of the control plane nodes. The best practice is to use a VIP, allowing for HA due to the fact that it is automatically assigned to another control plane node if one goes offline (https://www.talos.dev/v1.8/talos-guides/network/vip/)
    - You can use a hostname that points to the VIP, but I prefer to use the VIP directly.
    - Adding a VIP is essentially done by configuring `machine.network.interfaces.vip.ip` in the Talos machine config (see below), which automatically enables the functionality in Talos to load balance this VIP (e.g. if the VIP is on node 1 and that goes down, the VIP will automatically be moved to another node that is online).
    - It's not recommended to use the VIP for the Talos API (used with `talosctl`). The main purpose of the VIP is for use with the Kubernetes API (used with `kubectl`).
    - If you don't use a VIP, you can just use the IP of one of the control plane nodes, but if that particular node goes down, even if other control plane nodes are still up, you will not be able to run `kubectl` commands against your cluster without manually editing your `kubeconfig`. A VIP would also be pointless if you only have a single control plane node.
  - Feel free to change the cluster name (default is "talos-proxmox-cluster")
    ```bash
    # CONTROL_PLANE_IP should be the VIP you plan on using (not the DHCP assigned), or the IP of one of your control plane nodes if not using a VIP
    export CONTROL_PLANE_IP=10.0.50.160
    # FACTORY_IMAGE should be updated to the image you want to install
    export FACTORY_IMAGE=factory.talos.dev/installer/88d1f7a5c4f1d3aba7df787c448c1d3d008ed29cfb34af53fa0df4336a56040b:v1.9.2
    talosctl gen config talos-proxmox-cluster https://$CONTROL_PLANE_IP:6443 --with-secrets secrets.yaml --output-dir _out --install-image $FACTORY_IMAGE
    ```
- In Part 2, I discuss encrypting YAML files with SOPS + age, which you can use to encrypt this file and store it securely in a git repo: https://blog.dalydays.com/post/kubernetes-homelab-series-part-2-sops-and-age/ - Feel free to do that now if you want to add a little bit of complexity. Otherwise, continue to the end of this one and circle back to that article afterward so you can safely encrypt and store your configs in a git repo.
- Create node specific patches (used for defining a static IP, etc.): https://www.talos.dev/v1.8/talos-guides/configuration/patching/
  - Best practice is to install the base config to each node (`controlplane.yaml` or `worker.yaml`), then apply patches to customize nodes to set the name, static IP, etc.
  - To set a static IP, you have to specify either an `interface` or a `deviceSelector` (mutually exclusive). By default, Predictable Interface Names are not enabled, meaning your ethernet name might be something like `enxSOMETHING` based on the MAC address. One way around this is to enable Predictable Interface Names by setting the kernel argument `net.ifnames=0` on first boot, or you can use a generic `deviceSelector` like I did in the example below. If you only have 1 NIC, this will work just fine. If you mave multiple NICs, you can run `talosctl get links -n [node-ip]` and select the interface name from the ID column, or maybe the MAC address from the HW ADDR column. I'm just using `busPath: "0*"` to select the default NIC without changing any other options.
    - https://www.talos.dev/v1.8/talos-guides/network/predictable-interface-names/#single-network-interface
  - `mkdir -p patches`
  - Create patches for each controlplane and worker node:
    - cp1.patch
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
    - cp2.patch
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
    - cp3.patch
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
    - wk1.patch
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
    - wk2.patch
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
    - wk3.patch
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

## Talos Linux Installation
Finally, time to actually install Talos!

- Check the console for Talos CP1 in Proxmox to get the IP address (should have been assigned via DHCP). Let's say it's 10.0.50.129
- Install using base controlplane.yaml config: `talosctl apply-config --insecure --nodes 10.0.50.129 --file _out/controlplane.yaml`
- Follow this same process for each of the other 2 controlplane nodes.
- Follow the same process for all worker nodes, but use `_out/worker.yaml`: `talosctl apply-config --insecure --nodes 10.0.50.132 --file _out/worker.yaml`
- Watch the console in Proxmox to see it install and reboot. When you see the Kubernetes version and Kubelet status Healthy on all 6 nodes, you can proceed to patching each node to assign static IPs.
- Patch each node using the corresponding patch file. Endpoint should be the control plane node you're targeting, for controlplane setup. For workers, the endpoint needs to be one of the control plane nodes, and since you've already updated to 10.0.50.161 in this example, you can use the first control plane as the endpoint when patching all the worker nodes:
  -  `talosctl patch mc --talosconfig _out/talosconfig -e 10.0.50.129 -n 10.0.50.129 --patch @patches/cp1.patch`
  -  `talosctl patch mc --talosconfig _out/talosconfig -e 10.0.50.130 -n 10.0.50.130 --patch @patches/cp2.patch`
  -  `talosctl patch mc --talosconfig _out/talosconfig -e 10.0.50.131 -n 10.0.50.131 --patch @patches/cp3.patch`
  -  `talosctl patch mc --talosconfig _out/talosconfig -e 10.0.50.161 -n 10.0.50.132 --patch @patches/wk1.patch`
  -  `talosctl patch mc --talosconfig _out/talosconfig -e 10.0.50.161 -n 10.0.50.133 --patch @patches/wk2.patch`
  -  `talosctl patch mc --talosconfig _out/talosconfig -e 10.0.50.161 -n 10.0.50.134 --patch @patches/wk3.patch`
- Note: the VIP doesn't come online until after bootstrapping the cluster. Don't be like me and try to troubleshoot this right now :)
- If you want to deploy metrics-server (see https://www.talos.dev/v1.8/kubernetes-guides/configuration/deploy-metrics-server/) then enable `rotate-server-certificates: true` on all nodes and add the extra manifests to automatically approve the CSRs and include metrics-server. This is OPTIONAL.
  - Create `./patches/kubelet-cert-rotation.patch`
  ```yaml
  ---
  machine:
    kubelet:
      extraArgs:
        rotate-server-certificates: true
  
  cluster:
    extraManifests:
      - https://raw.githubusercontent.com/alex1989hu/kubelet-serving-cert-approver/main/deploy/standalone-install.yaml
      - https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
  ```
  -  `talosctl patch mc --talosconfig _out/talosconfig -e 10.0.50.161 -n 10.0.50.161 --patch @patches/kubelet-cert-rotation.patch`
  -  `talosctl patch mc --talosconfig _out/talosconfig -e 10.0.50.161 -n 10.0.50.162 --patch @patches/kubelet-cert-rotation.patch`
  -  `talosctl patch mc --talosconfig _out/talosconfig -e 10.0.50.161 -n 10.0.50.163 --patch @patches/kubelet-cert-rotation.patch`
  -  `talosctl patch mc --talosconfig _out/talosconfig -e 10.0.50.161 -n 10.0.50.171 --patch @patches/kubelet-cert-rotation.patch`
  -  `talosctl patch mc --talosconfig _out/talosconfig -e 10.0.50.161 -n 10.0.50.172 --patch @patches/kubelet-cert-rotation.patch`
  -  `talosctl patch mc --talosconfig _out/talosconfig -e 10.0.50.161 -n 10.0.50.173 --patch @patches/kubelet-cert-rotation.patch` 
- Configure talosctl by either copying talosconfig to $HOME/.talos/config or exporting the ENV variable:
  - `mv _out/talosconfig ~/.talos/config`
  - Or if you want to use ENV variables: `export TALOSCONFIG="_out/talosconfig"`
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
- Similar to talosctl, configure kubectl by either copying kubectl to $HOME/.kube/config or exporting the ENV variable:
  - `cp kubeconfig ~/.kube/config`
  - OR
  - `export KUBECONFIG="/root/whereami/kubeconfig"`
- Verify you can see the nodes. It can take a few minutes for Kubernetes to fully bootstrap, even though Talos shows Ready/Healthy in the dashboard.
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
- In testing I noticed some stuff that's supposed to run on all control plane nodes was only running on all control plane nodes. Give it some time and check again. You would be looking for `kube-apiserver-[node]`, `kube-controller-manager-[node]`, and `kube-scheduler-[node]`.
  - `kubectl get po -n kube-system`

## Test A Workload
- Create deployment: `kubectl create deploy nginx-test --image nginx --replicas 1`
- Expose via NodePort service: `kubectl expose deploy nginx-test --type NodePort --port 80`
- Get the node the pod is running on: `kubectl get po -o wide`
  - Note which worker node from the Node column
- Get the port on that node: `kubectl get svc`
  - Note the larger port number on the right side in the Ports column, likely in the 31000 range
- In testing, my pod was deployed to taloswk3 which has IP 10.0.50.173, and the port was 31458
  - Visit http://10.0.50.173:31458 in the browser to confirm you see the "Welcome to nginx!" page
- Clean up test resources:
  - `kubectl delete svc nginx-test`
  - `kubectl delete deploy nginx-test`

# Ongoing Maintenance

## Client Certificate Expiration - IMPORTANT!!!
https://www.talos.dev/v1.8/talos-guides/howto/cert-management/

> Talos Linux automatically manages and rotates all server side certificates for etcd, Kubernetes, and the Talos API. Note however that the kubelet needs to be restarted at least once a year in order for the certificates to be rotated. Any upgrade/reboot of the node will suffice for this effect.

TLDR; Your `talosconfig` and `kubeconfig` files are going to expire 1 year from when they are created. You need to regenerate these files, ideally **before** they expire to avoid issues connecting to the Talos API or the Kubernetes API.

Here's what you need to do. If you get lost, refer to the official Talos documentation linked above for curent instructions.

### talosconfig
You will know if `talosconfig` is expired if you see this error message when attempting to run `talosctl` commands. Wow that's a lot of errors!!!
```
error reading file: rpc errror: code = Unavailable desc = last connection error: connection error: desc = "error reading server preface: remote error: tls: expired certificate"
```

- At least once a year, regenerate `talosconfig`. You could automate this.
- Check config details with `talosctl config info`
  - If your current `talosconfig` is still valid: `talosctl -n CP1 config new talosconfig-reader --roles os:reader --crt-ttl 24h`
    - Here, `CP1` is the IP of one of your control plane nodes, e.g. `10.0.50.161`
  - If `talosconfig` has expired:
    - Generate from `secrets` bundle: `talosctl gen config --with-secrets secrets.yaml --output-types talosconfig -o talosconfig <cluster-name> https://<cluster-endpoint>`
      - Here, `<cluster-name>` should match what you named your Talos cluster earlier. `<cluster-endpoint>` should be the VIP used for your Talos cluster, e.g. 10.0.50.160. If you didn't elect to use VIP, just pick one of your control plane IPs.
  - There's a third option in the docs if you need it.
- If you store `talosconfig` in version control, re-encrypt with SOPS and update. This is totally optional and probably unnecessary since you can generate the config by other means if needed.
- `mv talosconfig ~/.talos/config`
- Configure your endpoints and nodes again since they are tied to the config:
  - `talosctl config endpoint 10.0.50.161 10.0.50.162 10.0.50.163`

### kubeconfig
You will know if `kubeconfig` is expired if you see this error message when attempting to run `kubectl` commands:
```
error: You must be logged in to the server (the server has asked for the client to provide credentials)
```
- At least once a year, regenerate `kubeconfig`. This requires you to have a valid `talosconfig`.
  - Follow the same steps as before to grab `kubeconfig`: `talosctl kubeconfig -n 10.0.50.161 .`
  - `mv kubeconfig ~/.kube/config`

## Backing Up `etcd` - Recommended!
I'll be talking about other backups in more depth later on in part 7 with Velero.

`etcd` is special as it's a real time representation of all Kubernetes cluster resources, and it's what the API queries when you run `kubectl` commands. It's the current running state of the cluser. Obviously backing this up is important, because if something goes wrong with it, and you don't have a backup, you will have to rebuild your entire cluster.

I strongly recommend backing up `etcd` separately. Since `etcd` doesn't run as part of the cluster, but rather at the Talos level, you can use `talosctl` to manage and backup the `etcd` database. This actually simplifies things from a Kubernetes administrator perspective, but it's certainly different than the approach you would take from the CKA certification training.

The basic command to snapshot etcd with Talos is this:
- `talosctl -n 10.0.50.161 etcd snapshot <path>`

I created a small bash script and scheduled it in a cron job to automate this backup, and my bash script looks like this. The resulting file looks like `etcd.snapshot.20241023`:
```
#!/bin/bash

TALOSCONFIG="/root/.talos/config"
PATH="/mnt/talos_backup"

/usr/local/bin/talosctl --talosconfig $TALOSCONFIG -n 10.0.50.161 etcd snapshot $PATH/etcd.snapshot.$(/bin/date +%Y%m%d)

# Delete snapshots older than 30 days
/usr/bin/find $PATH -mtime +30 -type f -delete
```

# Next Steps
Now you should have a Talos Linux cluster running with each node having its own static IP, along with a VIP for the control plane cluster. You are ready to start installing what I would consider foundational components that will be used to automate tasks for you when deploying actual workloads later on. These include:

- Secret encryption - Sealed Secrets
- LoadBalancer service - MetalLB
- Certificate manager - cert-manager
- Ingress provider - traefik
- CSI driver - democratic-csi
- Backups - Velero + etcd backup with talosctl
- Grafana (https://grafana.com/docs/grafana/latest/) + Prometheus (https://prometheus.io/docs/introduction/overview/)
- Loki + Promtail - https://github.com/grafana/loki
- Metrics server (https://github.com/kubernetes-sigs/metrics-server) - This is intended for autoscaling, not general monitoring