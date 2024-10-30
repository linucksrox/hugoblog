---
layout: blog
draft: false
title: Kubernetes Homelab Series Part 4 - Certificates With cert-manager and Let's Encrypt (WIP)
date: 2024-08-28
tags:
  - kubernetes
  - homelab
  - certificates
summary: A look into managing TLS certificates with cert-manager and Let's Encrypt, avoiding manual renewals.
---
# What is cert-manager?
https://cert-manager.io/  
cert-manager is a certificate controller for Kubernetes which is capable of handling all your certificate needs. This of course includes acquiring and automatically renewing certificates from Let's Encrypt, but it can also be used as a local CA (certificate authority) for private certificates between services, etc.

# Installation/Setup
https://cert-manager.io/docs/  
I run piHole internally on my network and also use DNS challenge for internal only hostnames. This means when I request a new certificate and cert-manager attempts to look up the DNS challenge record, it won't be able to query it through my piHole.

The solution is to configure cert-manager to specifically use public DNS servers to do the lookup, and that is done by setting `dns01-recursive-nameservers` and `dns01-recursive-nameservers-only`.

For that reason, I don't use the "Default static install" method by just installing the manifest using kubectl. Instead, I use the Helm chart so that I can apply those overrides during the installation.

If you don't need to override the nameservers used for DNS challenge, I would recommend using their `Default static install method`: https://cert-manager.io/docs/installation/#default-static-install

If you're still with me and want to apply using the Helm chart, follow along!

- Make sure you have Helm version 3 or later: https://helm.sh/docs/intro/install/
- Add the helm repo:
  ```
  helm repo add jetstack https://charts.jetstack.io --force-update
  ```
- Install the Helm chart. Note the `extraArgs` for DNS nameserver settings:
  ```
  helm install \
    cert-manager jetstack/cert-manager \
    --namespace cert-manager \
    --create-namespace \
    --version v1.16.1 \
    --set crds.enabled=true
    --set 'extraArgs={--dns01-recursive-nameservers-only,--dns01-recursive-nameservers=8.8.8.8:53\,1.1.1.1:53}'
  ```
  - If you're doing anything fancy on your network like me, double check that your Kubernetes nodes running cert-manager are able to reach the dns01-recursive-nameservers on port 53
- Verify deployment: `kubectl get all -n cert-manager`
- (optional) Verify dns01-recursive-nameserver settings were applied: `kubectl -n cert-manager describe deploy cert-manager | grep dns01`

At this point cert-manager should be running and ready to issue self-signed certificates. We can test that.
- Test with this file named `cert-manager-verify.yaml`:
  - Apply: `kubectl apply -f cert-manager-verify.yaml`
  - Verify: `kubectl describe cert -n cert-manager-test`
  - Cleanup: `kubectl delete -f cert-manager-verify.yaml`
  ```yaml
  apiVersion: v1
  kind: Namespace
  metadata:
    name: cert-manager-test
  ---
  apiVersion: cert-manager.io/v1
  kind: Issuer
  metadata:
    name: test-selfsigned
    namespace: cert-manager-test
  spec:
    selfSigned: {}
  ---
  apiVersion: cert-manager.io/v1
  kind: Certificate
  metadata:
    name: selfsigned-cert
    namespace: cert-manager-test
  spec:
    dnsNames:
      - example.com
    secretName: selfsigned-cert-tls
    issuerRef:
      name: test-selfsigned
  ```
Now we are ready to hook up Let's Encrypt so we can get certificates that are signed by a trusted authority. It's always a good idea to start with the staging issuer when testing Let's Encrypt so you don't run into rate limits with their production infrastructure.
- TODO complete steps to deploy staging issuer and test
- TODO complete steps to deploy production issuer and test