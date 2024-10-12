---
layout: blog
draft: false
title: Kubernetes Homelab Series Part 2 - Secrets With SOPS and age
date: 2024-10-12T13:12:09.310Z
tags:
  - kubernetes
  - homelab
  - secrets
summary: In this part we talk about encrypting secrets in key-value files like
  YAML so they can be stored securely in public places like GitHub.
---
# What Is SOPS and age?
https://github.com/getsops/sops  
SOPS is a tool to encrypt values in a key value format, which can be used with YAML. That means you can encrypt specific parts, leaving the YAML structure in tact and also make it safe to store this in a public place like GitHub.

https://github.com/FiloSottile/age  
(Pronounced "ah-gay", it does not rhyme with rage)  
This is a modern encryption tool which can be used to encrypt whole files, or in our use case used to encrypt parts of YAML files. SOPS supports other encryption options such as AWS KMS, most of which are cloud based. Since we are in a homelab, age seems to be a good option using a strong/modern encryption algorithm. The only other alternative for us homelabbers is PGP, but age is newer so it must be better, right?

# How Do I Use Them?
This is a quick walk through of getting SOPS and age installed and how to use SOPS to encrypt/decrypt YAML files. We'll do this for Talos Linux secrets.yaml and for talosconfig so they can be safely stored in a public repo.

# What about Sealed Secrets?
Sealed Secrets are a Kubernetes specific solution to encrypting secrets. It's not exactly a replacement for SOPS because it can't be used for anything outside the Kubernetes cluster, and `talosconfig` is a perfect example.

Sealed Secrets on the other hand are great for other things that you would normally put a Secret in Kubernetes. You can lock down access to the easily read base64 encoded Secrets in the cluster, making sure users interact only with Sealed Secrets which are truly encrypted. Sealed Secrets actually create regular Secrets in the cluster, so you still need to be careful about who has access to Secret resources.

You could use both SOPS and Sealed Secrets for different use cases, which is my plan going forward. Sometimes a Helm chart may not directly support Sealed Secrets and while it's possible to make it work, you may find it easier to use SOPS for that type of use case.
