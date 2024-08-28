---
layout: blog
draft: true
title: Kubernetes Homelab Series Part 2 - Secrets With SOPS and age
date: 2024-08-28T13:12:09.310Z
tags:
  - kubernetes
  - homelab
  - secrets
summary: In this part we talk about encrypting secrets in key-value files like
  YAML so they can be stored securely in public places like GitHub.
---
# What is SOPS?
https://github.com/getsops/sops  
SOPS is a way to encrypt secrets within a configuration file like YAML. It can encrypt the values but leave the file structure so that it's still human readable, while making it safe to store this in a public place like GitHub.

# What is age?
https://github.com/FiloSottile/age  
(Pronounced "ah-gay" and not age as in I'm getting a lot of gray hairs in my old age.)  
This is an encryption tool, the actual piece SOPS uses to do the encryption/decryption. SOPS supports other encryption options such as AWS KMS, most of which are cloud based. Since we are in a homelab, age seems to be a good option using a strong/modern encryption algorithm. The alternative for us homelabbers is PGP and trust me, age is a better choice for this use case.

# What about Sealed Secrets?
Elaborate on this, use cases and times you may prefer SOPS+age.
- Sealed Secrets is a controller on top of regular secrets. It actually creates regular secrets under the hood. One use case is if you set up RBAC, you can allow devs to read/write Sealed Secrets but not view regular secrets.
- SOPS+age can obviously be used outside of Kubernetes, which is important especially for bootstrapping stuff like Talos config, etc. It can also be useful for Helm deployments which don't directly support Sealed Secrets, rather than trying to adapt Sealed Secrets to work with it (need to clarify this one and I might be wrong on that point).
- You may end up using both for different purposes, which is totally fine.