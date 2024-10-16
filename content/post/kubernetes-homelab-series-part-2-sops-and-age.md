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

# Installation And Setup
This is a quick walk through of getting SOPS and age installed and how to use SOPS to encrypt/decrypt YAML files. We'll do this for Talos Linux secrets.yaml and for talosconfig so they can be safely stored in a public repo.

Install these tools whatever machine you would use to interact with your git repo. Upgrading is the same process as initially installing since we just pull the binary and copy into place.

## Installation
- Curl the latest `SOPS` binary and install
  - `curl -LJO https://github.com/getsops/sops/releases/download/v3.9.1/sops-3.9.1.x86_64.rpm`
  - `install -m 755 sops-v3.9.1.linux.amd64 /usr/local/bin/sops`
  - Verify sops version: `sops --version`
- Curl the latest `age` binary and install
  - `curl -LJO https://dl.filippo.io/age/latest?for=linux/amd64`
  - `tar zxvf age-v1.2.0-linux-amd64.tar.gz`
  - `install -m 755 age/age /usr/local/bin/age && install -m 755 age/age-keygen /usr/local/bin/age-keygen`
  - Verify age version: `age --version`
  - Verify age-keygen version: `age-keygen --version`

## Setup
- Create a private key similar to how you would with SSH
  - `mkdir -p ~/.config/sops/age`
  - `age-keygen -o ~/.config/sops/age/keys.txt`
  - Copy the private/public key somewhere such as in Bitwarden or other secure location
  - Copy down the public key, which is needed to encrypt. The public key is included in the private key file in case you need to reference it in the future.
- Create a config file locally `.sops.yaml` (usually stored and committed in the repo where it is used) which tells SOPS which files to encrypt, which keys to encrypt within the file, and which encryption to use. See the documentation for more details around the config file.
  - `path_regex` tells sops which files to encrypt if not specified
  - (optional) `encrypted_regex` defines which keys within the file to encrypt - omit this to encrypt all values
  - `age` is the public key used for encryption
  - Sample `.sops.yaml` file configured for Talos Linux secrets.yaml and talosconfig files:
```yaml
---
creation_rules:
  - path_regex: /*secrets(\.encrypted)?.yaml$
    encrypted_regex: "(^id|secret|bootstraptoken|secretboxencryptionsecret|token|ca|crt|key)$"
    age: age1.....(public age key goes here)
  - path_regex: /*talosconfig(\.encrypted)?$
    age: age1.....(public age key goes here)
```

# Encrypting and Decrypting Files
Now you are ready to actually encrypt/decrypt values in your files. This is a good point to come up with a naming convention if you want to keep separate copies of the decrypted files, because you should make sure you add the decrypted version to .gitignore since the whole point of this is to make sure you don't commit plaintext secrets to a repo anywhere (public or private).

- Encryption: `sops --encrypt secrets.yaml > secrets.encrypted.yaml` or `sops --encrypt talosconfig > talosconfig.encrypted`
- Decryption: `sops --decrypt secrets.encrypted.yaml > secrets.yaml` or `sops --decrypt talosconfig.encrypted > talosconfig`

# What about Sealed Secrets?
Sealed Secrets are a Kubernetes specific solution to encrypting secrets. It's not exactly a replacement for SOPS because it can't be used for anything outside the Kubernetes cluster, and `talosconfig` is a perfect example.

Sealed Secrets on the other hand are great for other things that you would normally put a Secret in Kubernetes. You can lock down access to the easily read base64 encoded Secrets in the cluster, making sure users interact only with Sealed Secrets which are truly encrypted. Sealed Secrets actually create regular Secrets in the cluster, so you still need to be careful about who has access to Secret resources.

You could use both SOPS and Sealed Secrets for different use cases, which is my plan going forward. Sometimes a Helm chart may not directly support Sealed Secrets and while it's possible to make it work, you may find it easier to use SOPS for that type of use case.
