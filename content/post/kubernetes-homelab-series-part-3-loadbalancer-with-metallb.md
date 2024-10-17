---
layout: blog
draft: true
title: Kubernetes Homelab Series Part 3 - LoadBalancer With MetalLB
date: 2024-08-28T13:20:32.765Z
tags:
  - kubernetes
  - homelab
summary: A dive into implementing the LoadBalancer service type without a cloud provider using MetalLB.
---
# What's a LoadBalancer?
One of the first things you might run into after building a new Kubernetes cluster is that as soon as you go to build something that has a LoadBalancer type service, it's stuck in a Pending status. That's because you don't have a "controller" (is that the right term?) to handle this type of service. It's not included out of the box.

## See For Yourself
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
- Check the service status with `kubectl get svc` and look at the EXTERNAL-IP for the `nginx-loadbalancer` service. It should look like

Why? Because it requires some environment specific configuration, and most importantly in our case that's a pool of IP addresses that can be used for this purpose. So how do we tell Kubernetes what IPs it can use for LoadBalancer services? MetalLB - https://metallb.io/

There are other options such as KubeVIP, which is also a great project. I'm going to use MetalLB because that's what I have already figured out. You could even use KubeVIP as a control plane load balancer, in addition to MetalLB as a service load balancer, but now I"m getting into the weeds.

# MetalLB Setup
Elaborate...