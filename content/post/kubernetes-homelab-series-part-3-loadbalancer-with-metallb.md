---
layout: blog
draft: false
title: Kubernetes Homelab Series Part 3 - LoadBalancer With MetalLB
date: 2024-10-18
tags:
  - kubernetes
  - homelab
summary: A dive into implementing the LoadBalancer service type without a cloud provider using MetalLB.
---
# What's a LoadBalancer?
There are a few different types of "Services" in Kubernetes. They provide solutions for different types of network communication between internal cluster resources and externally. Depending on what you need to connect to what, you will use a suitable Service type to provide that connection. I'm specifically not mentioning those other types because this isn't really a tutorial on Kubernetes or what Service type you might need. But you will most likely need/want a LoadBalancer service if you need external traffic to reach within your cluster.

One of the first things you might run into after building a new Kubernetes cluster is that as soon as you go to build something that has a LoadBalancer type service, it's stuck in a Pending status. That's because you don't have a "controller" (is that the right term?) to handle this type of service. It's not included out of the box.

Why not include LoadBalancer out of the box? Because it requires some environment specific configuration, and most importantly in our case that's a pool of IP addresses that are dedicated for this purpose. So how do we tell Kubernetes what IPs it can use for LoadBalancer services? MetalLB - https://metallb.io/ There's also the difference between ARP and BGP. I don't have a need for BGP in my homelab so I'm just using ARP which is the easier option.

There are other options such as KubeVIP (https://kube-vip.io/), which is also a great project. I'm going to use MetalLB because that's what I have already figured out. You could even use KubeVIP as a control plane load balancer, in addition to MetalLB as a service load balancer. One is dedicated to load balancing the control plane, which means you can set one IP address for all 3 control plane nodes as an interface between `kubectl` and your cluster's API. But what we're focused on here is Service load balancing, so the same thing but for all the stuff you're actually running on your cluster like websites, etc. And eventually this will tie into Ingress and Gateway API, but let's not get too far into the weeds yet.

## See For Yourself
If you want to see the behavior wen you create a LoadBalancer service without something that implements this type of service such as MetalLB, follow along. The symptom is that instead of getting an IP assigned, you will see the EXTERNAL-IP "stuck" as `<pending>`. 

- Save this as `test-lb-svc.yaml` and then run `kubectl apply -f test-lb-svc.yaml`. This creates a deployment running 2 replicas of nginx, then creates a LoadBalancer service in front of it. 
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-loadbalancer
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```
- Check the service status with `kubectl get svc` and look at the EXTERNAL-IP for the `nginx-loadbalancer` service. Notice the `<pending>` instead of an actual IP address:
```
NAME                 TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
kubernetes           ClusterIP      10.96.0.1       <none>        443/TCP        40h
nginx-loadbalancer   LoadBalancer   10.99.103.170   <pending>     80:30652/TCP   3s
```
- You can delete this and try again later after deploying MetalLB, or leave it and it will automatically get an IP assigned by MetalLB once it's running.

# MetalLB Setup
https://metallb.universe.tf/installation/

The core steps are to make sure your cluster networking will support MetalLB, then install MetalLB, then apply your environment specific configuration. Talos includes flannel which supports MetalLB, and no other networking changes are required before you just start installing. There is no configmap for kube-proxy either, that is already configured in Talos and will work fine.

- Install the MetalLB manifest (check the documentation for the current version): `kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.8/config/manifests/metallb-native.yaml`
- Configure your IP address pool for MetalLB to use. You can list multiple ranges if you want. See documentation for examples: https://metallb.io/configuration/#defining-the-ips-to-assign-to-the-load-balancer-services
```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 10.0.50.64/28
```
  - `kubectl apply -f metallb-ipaddresspool.yaml`
- Configure how MetalLB will announce new IPs to your network. See documentation for details or use this for layer 2 advertisement: https://metallb.io/configuration/#announce-the-service-ips
```yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: metallb-advertise
  namespace: metallb-system
```
  - `kubectl apply -f metallb-l2advertisement.yaml`

You should now have MetalLB running and ready to provision IP addresses for your Loadbalancer services. If you ran the demo earlier and left the service up, check again to verify that there is now an IP assigned to the service (see above for details on how to test and verify).
