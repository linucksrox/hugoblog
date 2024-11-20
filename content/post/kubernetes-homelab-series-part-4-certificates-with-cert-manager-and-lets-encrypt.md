---
layout: blog
draft: false
title: Kubernetes Homelab Series Part 4 - Certificates With cert-manager and Let's Encrypt (WIP)
date: 2024-11-19
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

I'm using Cloudflare for my DNS nameservers so these steps will be based on that setup.

- Create a Kubernetes secret resource with your Cloudflare API token. This allows cert-manager to add custom _acme-challenge records for domain validation.
  - In Cloudflare, you can generate a new API token by navigating to your user account > My Profile > API Tokens > Create Token. Make sure token permissions include `All zones - Zone Settings:Read, Zone:Read, DNS:Edit` or you can limit to a specific zone if you have multiple and want to use different API Tokens for different zones.
  ```yaml
  apiVersion: v1
  kind: Secret
  metadata:
    name: letsencrypt-cloudflare-api-token-secret
    namespace: cert-manager
  type: Opaque
  stringData:
    api-token: <api-token-goes-here>
  ```
- Apply the secret manifest: `kubectl apply -f letsencrypt-cloudflare-api-token-secret.yaml`  

cert-manager has two types of issuers: Issuer and ClusterIssuer. Issuer is used for more fine-grained control, it can be placed within a specific namespace, etc. ClusterIssuer, as the name implies, can be accessed across the whole Kubernetes cluster. Since I'm not getting too complicated, and also want to be able to easily utilize cert-manager from multiple namespaces, I'm using ClusterIssuer. If you wanted to use Issuer, just pick a namespace and the steps are almost identical to this.

- Create a manifest file for the staging issuer. Update your email address:
  ```yaml
  apiVersion: cert-manager.io/v1
  kind: ClusterIssuer
  metadata:
    name: letsencrypt-staging
  spec:
    acme:
      # The ACME server URL
      server: https://acme-staging-v02.api.letsencrypt.org/directory
      # Email address used for ACME registration
      email: mail@example.com
      # Name of a secret used to store the ACME account private key
      privateKeySecretRef:
        name: letsencrypt-staging-issuer-secret
      # Enable the DNS-01 challenge provider
      solvers:
      - dns01:
          cloudflare:
            apiTokenSecretRef:
              name: letsencrypt-cloudflare-api-token-secret
              key: api-token
  ```
- Apply the staging issuer: `kubectl apply -f letsencrypt-staging-clusterissuer.yaml`
- Verify your ClusterIssuer exists (or Issuer if that's what you're using): `kubectl get clusterissuer`

Now we need to deploy a Certificate resource (again) to test our Let's Encrypt staging ClusterIssuer. This is similar to the verify manifest like before, but only needs to include the Certificate resource. However, we also need to update a couple things, notably adding the field Certificate.spec.issuerRef.kind and setting its value to ClusterIssuer (see https://cert-manager.io/docs/usage/certificate/)
```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: letsencrypt-staging-certificate
  namespace: default
spec:
  dnsNames:
    - justkidding.example.com
  secretName: letsencrypt-staging-cert-tls
  issuerRef:
    kind: ClusterIssuer
    name: letsencrypt-staging
```
- Apply the Certificate manifest: `kubectl apply -f letsencrypt-staging-certificate.yaml`
- Be patient. This can sometimes be quick, but sometimes can take a while. My most recent attempt took 34 minutes (see Troubleshooting section below to dive into what's happening during the process).
- Check the Certificate is ready: `kubectl get certificate` - this will show Ready=False until it's fully issued
- Troubleshooting steps can involve looking at:
  - Definitive guide: https://cert-manager.io/docs/troubleshooting/
    - Also for Let's Encrypt: https://cert-manager.io/docs/troubleshooting/acme/
  - `kubectl get certificaterequests`
  - `kubectl describe certificaterequests`
  - `kubectl get order`
  - `kubectl describe order`
  - `kubectl get challenge`
  - `kubectl describe challenge`
    - This level will tell you if it's stuck waiting for DNS challenge (basically it's waiting for the _acme-challenge record to be added and validated from the cert-manager service. This is where sometimes you can get timeouts if your network doesn't allow cert-manager to query public DNS servers to check).
    - Troubleshooting steps I've had here were basically verifying that the DNS record exists in Cloudflare while it's waiting. If it is, wait a while to see if it eventually solves the challenge. If not, you might need to dive deeper into dns01-recursive-nameserver settings (see above) or otherwise make sure your internal DNS resolver is able to query the new record.
    - In the output, look for Challenge.Spec.Key which is the value you should see in the _acme-challenge TXT record
  - You can also check the logs on the cert-manager container
    - Find the pod name: `kubectl get po -n cert-manager`
    - `kubectl logs cert-manager-556766675f-pt123 -n cert-manager`
  - See cert-manager issue for more discussion: https://github.com/cert-manager/cert-manager/issues/5917

- TODO complete steps to deploy production issuer and test