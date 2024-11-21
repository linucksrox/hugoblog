---
layout: blog
draft: false
title: Kubernetes Homelab Series Part 4.5 - Debug Pod
date: 2024-11-20
tags:
  - kubernetes
  - homelab
  - debug
summary: A quick look at running a debug pod in lieu of SSH for Talos Linux.
---
# Why is this needed?
If you're like me and you have a habit of SSHing into an environment to troubleshoot an issue, you may be at a loss with Talos Linux since one does not simply SSH into a Talos node. Lord Of The Rings reference right there :)

What you can do is run a "debug" container with elevated privileges, in the kube-system namespace, which essentially gives you roughly the same thing. This is how.

Real quick, Talos themselves wrote an article on this including recommendations for alternative methods. I'm forging ahead the "old" way, but don't discredit the new approach either as this type of thing is likely going to be the future and we need to adapt! Talos offers a lot of excellent troubleshooting tools via their talosctl API and if all you need is something like netstat, they've got you covered.

https://www.siderolabs.com/blog/how-to-ssh-into-talos-linux/

On the other hand, I just needed to test manually mounting NVMe over TCP when democratic-csi wasn't working as expected, and this was the only way :)

# Show Me The Money!
Anyway, here's how to deploy the debug container.

- Write a new file `debugpod.yaml` and replace [nodename] with the name of the specific node you want this deployed to in your cluster. Optionally change the image if you don't want to use Alpine. This is my go-to default because of how small and fast it is, and installing packages with APK is about as fast as it gets.
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: debugpod
    namespace: kube-system
  spec:
    hostPID: true
    containers:
    - name: debugcontainer
      image: alpine:3.20
      stdin: true
      tty: true
      securityContext:
        privileged: true
      volumeMounts:
      - name: dev-mount
        mountPath: /dev
    volumes:
    - name: dev-mount
      hostPath:
        path: /dev
    nodeSelector:
      kubernetes.io/hostname: [nodename]
  ```
- Deploy the pod: `kubectl apply -f debug-pod.yaml`
- Drop into a shell inside the container: `kubectl exec -it debugpod -n kube-system -- /bin/sh`
- Do something useful. In my case, I wanted to check if a pod running in this cluster can reach out to Cloudflare and Google DNS servers so:
  - Install telnet: `apk add --update busybox-extras`
  - `telnet 1.1.1.1 53` - Ctrl+C, then e to exit
  - `telnet 8.8.8.8 53` - Ctrl+C, then e to exit
- Whatever else you want to test :) For example, check this democratic-csi issue for what I did from here to test nvmeof mounts: https://github.com/democratic-csi/democratic-csi/issues/418