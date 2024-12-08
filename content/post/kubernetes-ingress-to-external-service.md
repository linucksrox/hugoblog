---
layout: blog
draft: false
title: Kubernetes - External Services And Ingress
date: 2024-12-08
tags:
  - kubernetes
  - homelab
  - ingress
summary: A quick overview of using Ingress to proxy to a service that doesn't live in the Kubernetes cluster.
---
# About External Services
External services are anything you want to route traffic to that does not live in your Kubernetes cluster. For example, you might be running MinIO in a VM and accessing it via IP address, but you want to basically reverse proxy to it. You can use Kubernetes Ingress to act as a reverse proxy to pretty much anything, even if it doesn't live within your cluster.

It's pretty straightforward to do this, and you just need to create a few resources: `Service`, `EndpointSlice` and `Ingress`. For the sake of organization, I like to use a separate namespace to separate any external service resources. If you're doing this, just create a new namespace as your first step.

# MinIO Example
I have MinIO running outside my Kubernetes cluster. For now, I just want basically a reverse proxy in front of it, but I want to use Kubernetes Ingress since I already have that running and configured to get TLS certificates from Let's Encrypt.

- Optionally, create a new `Namespace`. I'm using "externalservices"
  ```yaml
  apiVersion: v1
  kind: Namespace
  metadata:
    name: externalservices
  ```
- Ceate an `EndpointSlice`. This object is the newer replacement for `Endpoints` which are now managed by `EndpointSlice`, so it's no longer recommended to manually create `Endpoints` directly. `EndpointSlice` requires a name and a label that matches the service name you will be using. The label's key is `kubernetes.io/service-name`. `namespace` is optional but recommended. Then you just specify address type (IPv4 or IPv6), ports, and the actual endpoint with the IP address. Here's an example of my EndpointSlice manifest where MinIO listening at 192.168.1.35 on port 80. I'm using a namespace called "externalservices" and the service name is minio-service.
  ```yaml
  apiVersion: discovery.k8s.io/v1
  kind: EndpointSlice
  metadata:
    name: minio-service
    namespace: externalservices
    labels:
      kubernetes.io/service-name: minio-service
  addressType: IPv4
  ports:
    - port: 80
  endpoints:
    - addresses:
      - "192.168.1.35"
      conditions:
        ready: true
  ```
- Next, create a ClusterIP `Service` without a selector. Having no selectors allows the service to be attached directly to an Endpoint that you create instead of only being able to attach to a pod. The way it knows how to attach to the EndpointSlice is based on the label you used in the EndpointSlice manifest which has to exactly match the name you use in this service (in this case minio-service). This service just maps port 80 to targetPort 80, and we will handle HTTP to HTTPS redirection, plus TLS termination in the Ingress resource.
  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: minio-service
    namespace: externalservices
  spec:
    type: ClusterIP
    ports:
      - port: 80
        protocol: TCP
        targetPort: 80
  ```
- Finally, create an `Ingress`. Here's a basic example using a subdomain, using a Let's Encrypt ClusterIssuer, and using traefik as the ingress class.
  ```yaml
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: minio-service-ingress
    namespace: externalservices
    annotations:
      cert-manager.io/cluster-issuer: "letsencrypt-staging"
      traefik.ingress.kubernetes.io/router.entrypoints: websecure
  spec:
    ingressClassName: traefik
    tls:
    - hosts:
      - minio.example.com
      secretName: tls-example-com
    rules:
    - host: minio.example.com
      http:
        paths:
        - backend:
            service:
              name: minio-service
              port:
                number: 80
          path: /
          pathType: Prefix
  ```

TLDR; Put it all together into a single manifest:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: externalservices
---
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: minio-service
  namespace: externalservices
  labels:
    kubernetes.io/service-name: minio-service
addressType: IPv4
ports:
  - port: 80
endpoints:
  - addresses:
    - "192.168.1.35"
    conditions:
      ready: true
---
apiVersion: v1
kind: Service
metadata:
  name: minio-service
  namespace: externalservices
spec:
  type: ClusterIP
  ports:
    - port: 80
      protocol: TCP
      targetPort: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: minio-service-ingress
  namespace: externalservices
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-staging"
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
spec:
  ingressClassName: traefik
  tls:
  - hosts:
    - minio.example.com
    secretName: tls-example-com
  rules:
  - host: minio.example.com
    http:
      paths:
      - backend:
          service:
            name: minio-service
            port:
              number: 80
        path: /
        pathType: Prefix
```
- Apply `kubectl apply -f minio-external-service.yaml`
- Shortly after deploying the resources, update DNS or your local hosts file and point to minio.example.com. It may take a bit to generate a signed certificate from the issuer, but you should still be able to route to your endpoint through the Ingress and verify that it's working.

That's it. That's the end. You are now done.