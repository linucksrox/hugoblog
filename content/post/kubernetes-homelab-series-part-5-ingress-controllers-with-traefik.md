---
layout: blog
draft: true
title: Kubernetes Homelab Series Part 5 - Ingress Controllers With Traefik
date: 2024-11-21
tags:
  - kubernetes
  - homelab
  - ingress
  - traefik
summary: A look into building an ingress controller and the purpose behind it.
---
# What Is Ingress?
Ingress is a way to expose your applications running in Kubernetes to something outside the cluster. It's like a reverse proxy for Kubernetes. It defines how traffic is routed from the point that it hits the cluster, specifically the ingress controller, to the running pod. You might already know about the different Kubernetes service types such as ClusterIP, NodePort and LoadBalancer, but this takes it to another level that can handle TLS termination and advanced routing rules, among other nice features, depending on which Ingress Controller you pick. That determines exactly which features are available.

# What's An Ingress Controller?
An ingress controller is the software that contains the actual logic to implement Ingress rules and functionality. These can vary depending on the ingress controller you pick.

## What About NGINX?
I feel like I should mention that one of the most widely known ingress controllers is nginx. Before you go down that rabbit hole, there are 2 different ingress controllers based on nginx. One used to be called `nginx-ingress` which was/is the official open source nginx ingress controller (now it's `kubernetes-ingress`: https://github.com/nginxinc/kubernetes-ingress), and maintained by the engineers at NGINX. Hard to go wrong there. The other popular option is `ingress-nginx` which is the community maintained, NGINX supported project. It's been a while since I tried either one, but I hear good things about ingress-nginx because there's a large community behind it and it should be easier to find answers if you have questions. If you need something more production ready (and have your heart set on NGINX), then maybe consider `kubernetes-ingress`.

# Why Traefik?
A valid question you might ask, and I've asked myself, is why would I consider using Traefik instead of many people's go-to NGINX? These are my reasons:
- I settled on Traefik years ago when it was still v1 for things I was running in Docker. It provided the ability to EASILY reverse proxy to my applications by simply setting a few labels in my docker-compose.yml files, and INCLUDING the ability to automatically get certificates from Let's Encrypt using DNS-01 challenge (offering legitimate certificates even for internal only applications).
- It's open-source, it's fast, and it just works (I've never had issues or run into bugs personally using Traefik)
- When I first went to Traefik, it had more features than NGINX. Notably it can dynamically add routes/rules without restarting or reloading. Maybe the NGINX controller functions the same way today, but in the Docker compose world that was not the case.
- Native support for HTTP/3 - it seems to be more on the cutting edge, supporting new technologies faster than NGINX. I know - newer isn't always better, just look at Debian. But for my homelab, newer is almost always better :)
- Middleware - more features readily available such as IP whitelisting, WAF options, etc. that you can easily plug into routes for additional functionality

## Why Not NGINX Or Kong Or Something Else?
No real reason. I like the Traefik project and it does what I need in my homelab, so I will continue using it for now :)

# Installation/Setup
https://doc.traefik.io/traefik/getting-started/install-traefik/#use-the-helm-chart



# What About Gateway API?
This Kong article has a lot of good information on the topic: https://konghq.com/blog/engineering/gateway-api-vs-ingress

I have read that Gateway API is the successor to ingress controllers. The official FAQ says that Gateway API will not replace Ingress, so we are safe to continue using them (https://gateway-api.sigs.k8s.io/faq/#will-gateway-api-replace-the-ingress-api). There is no need to use Gateway API if all you need is Ingress.

Gateway API solves some inherent limitations with Ingress, primarily the fact that Ingress is Layer 7 only.

## Gateway API Installation/Setup - And Migrating From Ingress
How to migrate from Ingress to Gateway API: https://gateway-api.sigs.k8s.io/guides/migrating-from-ingress/