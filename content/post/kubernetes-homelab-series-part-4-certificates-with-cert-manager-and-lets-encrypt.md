---
layout: blog
draft: true
title: Kubernetes Homelab Series Part 4 - Certificates With cert-manager and
  Let's Encrypt
date: 2024-08-28T13:32:43.305Z
tags:
  - kubernetes
  - homelab
  - certificates
summary: A look into managing TLS certificates with cert-manager and Let's
  Encrypt, avoiding manual renewals.
---
# What is cert-manager?
https://cert-manager.io/  
cert-manager is a certificate controller for Kubernetes which is capable of handling all your certificate needs. This of course includes acquiring and automatically renewing certificates from Let's Encrypt, but it can also be used as a local CA (certificate authority) for private certificates between services, etc.

# Installation/Setup
- f