---
layout: blog
draft: false
title: Kubernetes Homelab Series Part 5 - Ingress Controllers With Traefik
date: 2024-11-23
tags:
  - kubernetes
  - homelab
  - ingress
  - traefik
summary: A look into building an ingress controller and the purpose behind it.
---
# What Is Ingress and Ingress Controller?
Ingress is a way to expose your applications running in Kubernetes to something outside the cluster. It's like a reverse proxy for Kubernetes. It defines how traffic is routed from the point that it hits the cluster, specifically the ingress controller, to the running pod. You might already know about the different Kubernetes service types such as ClusterIP, NodePort and LoadBalancer, but this takes it to another level that can handle TLS termination and advanced routing rules, among other nice features, depending on which Ingress Controller you pick. That determines exactly which features are available.

An ingress controller is the software that contains the actual logic to implement Ingress rules and functionality. These can vary depending on the ingress controller you pick.

The awesome thing about this component is that it deploys a LoadBalancer type service, and if you used MetalLB or kube-vip, that means you get a virtual IP dynamically assigned to one of your ingress controller replicas. That means if the pod goes down that's running the ingress controller (assuming you have more than 1 replica or use a DaemonSet) then the IP automatically moves to another replica. In other words, between MetalLB and Traefik (or insert your tool of choice for either), you get highly available Ingress to your Kubernetes cluster with no manual effort. Pretty nifty!

## What About NGINX?
I feel like I should mention that one of the most widely known ingress controllers is nginx. Before you go down that rabbit hole, there are 2 different ingress controllers based on nginx. One used to be called `nginx-ingress` which was/is the official open source nginx ingress controller (now it's `kubernetes-ingress`: https://github.com/nginxinc/kubernetes-ingress), and maintained by the engineers at NGINX. Hard to go wrong there. The other popular option is `ingress-nginx` which is the community maintained, NGINX supported project. It's been a while since I tried either one, but I hear good things about ingress-nginx because there's a large community behind it and it should be easier to find answers if you have questions. If you need something more production ready (and have your heart set on NGINX), then maybe consider `kubernetes-ingress`.

# Why Traefik?
A valid question you might ask, and I've asked myself, is why would I consider using Traefik instead of many people's go-to NGINX? These are my reasons:
- I settled on Traefik years ago when it was still v1 for things I was running in Docker. It provided the ability to EASILY reverse proxy to my applications by simply setting a few labels in my docker-compose.yml files, and INCLUDING the ability to automatically get certificates from Let's Encrypt using DNS-01 challenge (offering legitimate certificates even for internal only applications).
- It's open-source, it's fast, and it just works (I've never had issues or run into bugs personally using Traefik)
- When I first went to Traefik, it had more features than NGINX. Notably it can dynamically add routes/rules without restarting or reloading. Maybe the NGINX controller functions the same way today, but in the Docker compose world that was not the case.
- Native support for HTTP/3 - it seems to be more on the cutting edge, supporting new technologies faster than NGINX. I know - newer isn't always better, just look at Debian. But for my homelab, newer is almost always better :)
- Middleware - more features readily available such as IP whitelisting, WAF options, etc. that you can easily plug into routes for additional functionality

In Kubernetes it matters a lot less since the Ingress resource is another layer of abstraction, and either way your stuff will get routed. But at some point you may have specific requirements, certain middlewares functionality, performance requirements, etc. and that may drive the decision to go with whatever tool best meets those requirements.

## Why Not NGINX Or Kong Or Something Else?
I have no argument against using NGINX or anything else. I just like the Traefik project and it does what I need in my homelab, so I will continue using it for now :)

# Installation/Setup
You can install the helm chart with no extra options and it will "just work." There are also some good options you might want to at least know about. These can also be changed later on, so it's not a big deal if you change your mind later.

https://doc.traefik.io/traefik/getting-started/install-traefik/#use-the-helm-chart

## Vanilla Installation
- For any helm installation, always start by adding the helm repo:
  - `helm repo add traefik https://traefik.github.io/charts`
  - `helm repo update`
- Vanilla Installation:
  - `helm install traefik traefik/traefik`

You can start deploying Ingress resources. But keep reading to understand some of the ingress controller options and why you might consider changing them.

## Deployment Vs. DaemonSet
By default the Helm chart creates a Deployment with 1 replica, based on the [values.yml file](https://github.com/traefik/traefik-helm-chart/blob/master/traefik/values.yaml). You could scale this up to 3 replicas if you're looking for something more highly available, or you could consinder a DaemonSet where there will always be one Traefik instance on each worker node to handle routing. Keep in mind with a DaemonSet that if you ever scale up the number of worker nodes, that will also scale up the number of total Ingress Controller pods.

If you wanted to do something a little more elaborate like only run 3 replicas, but make sure they never run on the same node as each other (3 different physical nodes), you would need to get into some node affinity rules that I won't explain here because I haven't done that (yet).

When customizing Helm charts, I tend to prefer downloading the values.yaml file, customizing it, then deploying using the custom values file. That way I can store that in version control and not have to think about messy JSON stuff in my shell commands.
- Download values file: `helm show values traefik/traefik > values.yaml`
- Customize. Let's change to a DaemonSet by changing deployment.kind from Deployment to DaemonSet. Capitalization matters here! DaemonSet with a capital S:
  ```yaml
  ...
  deployment:
    # -- Enable deployment
    enabled: true
    # -- Deployment or DaemonSet
    kind: DaemonSet
  ...
  ```

- Let's say you have 10 worker nodes and you want Traefik to be replicated, but you don't need 10 instances running. In that case you could do a Deployment and set the number of replicas to 3.
  ```yaml
  ...
  deployment:
    # -- Enable deployment
    enabled: true
    # -- Deployment or DaemonSet
    kind: Deployment
    replicas: 3
  ...
  ```
- Deploy: `helm install traefik traefik/traefik -f values.yaml`

## Namespace
Another consideration when deploying Traefik is whether you want to use a custom namespace. Namespaces are a best practice instead of dumping everything into the default namespace for security and organization. It's easy enough to split out Traefik, so why not?

- Create a namespace: `kubectl create ns traefik`
- Deploy: `helm install traefik traefik/traefik -f values.yaml -n traefik`

OR do it all in a single step by adding `--create-namespace`
- `helm install traefik traefik/traefik -f values.yaml -n traefik --create-namespace`

## Verify Your Deployment (or DaemonSet)
- Vanilla install: `kubectl get deploy`
  ```
  NAME      READY   UP-TO-DATE    AVAILABLE   AGE
  traefik   1/1     1             1           3m
  ```
- Namespaced Deployment: `kubectl get deploy -n traefik`
  ```
  NAME      READY   UP-TO-DATE    AVAILABLE   AGE
  traefik   3/3     3             3           14s
  ```
- Namespaced DaemonSet: `kubectl get daemonset -n traefik`
  ```
  NAME      DESIRED   CURRENT   READY   UP-TO-DATE    AVAILABLE   NODE SELECTOR   AGE
  traefik   3         3         3       3             3           <none>          12s
  ```

## Traefik Dashboard
Traefik has a cool dashboard showing some details about routes, middlewares, services, and other stuff. It's all read-only, but also nice to look at and sometimes helpful in troubleshooting why a route isn't working as expected or what exactly it's routing to. But you have to enable access to it, and consider what type of security you want to have in place for Dashboard access.

They don't recommend enabling the dashboard in production, but if you do, at least secure it. The easiest way to expose but also keep somewhat secure is using basic authentication. Sounds good enough for me, let's try it out!

https://github.com/traefik/traefik-helm-chart/blob/master/EXAMPLES.md#publish-and-protect-traefik-dashboard-with-basic-auth

- You can throw this in its own yaml file and apply this specific config to the existing installation. Let's call this `dashboard-basicauth.yaml`:
  - You'll need a DNS record pointing the host in the matchRule to the IP address on Traefik's LoadBalancer service.
  - Using websecure without specifying TLS options will result in a self-signed certificate and you'll get a warning in the browser.
  - Changing these things is beyond the scope of this tutorial - this is where I draw the line :)
```yaml
# Create an IngressRoute for the dashboard
ingressRoute:
  dashboard:
    enabled: true
    # Custom match rule with host domain
    matchRule: Host(`traefik-dashboard.example.com`)
    entryPoints: ["web","websecure"]
    # Add custom middlewares : authentication and redirection
    middlewares:
      - name: traefik-dashboard-auth

# Create the custom middlewares used by the IngressRoute dashboard (can also be created in another way).
# /!\ Yes, you need to replace "changeme" password with a better one. /!\
extraObjects:
  - apiVersion: v1
    kind: Secret
    metadata:
      name: traefik-dashboard-auth-secret
    type: kubernetes.io/basic-auth
    stringData:
      username: admin
      password: changeme

  - apiVersion: traefik.io/v1alpha1
    kind: Middleware
    metadata:
      name: traefik-dashboard-auth
    spec:
      basicAuth:
        secret: traefik-dashboard-auth-secret
```
- Apply the config: `helm upgrade --reuse-values -n traefik -f dashboard-basicauth.yaml traefik traefik/traefik`

Note that the ingressRoute you set up in this config will NOT appear in your cluster as an Ingress type resource. So if you run `kubectl get ingress -A` you won't find it and can't verify it that way. It's an internal ingress route that Traefik uses.

## HTTP Redirect To HTTPS
This is another common thing that most people probably do. You can do this on a per-service basis, or configure Traefik to automatically redirect HTTP to HTTPS for everything. In that case, if you happen to have something that you don't want redirected, you would need to create another entrypoint and use a different port besides 80.

https://doc.traefik.io/traefik/routing/entrypoints/#redirection

I'm confused about the mismatch between the values.yaml file and the documentation. In values.yaml there's a field called `redirectTo` that expects an object. I worked out that it requires a `port` and possibly can take another field named `permanent` (true/false). However, when I upgraded the helm release with these values it didn't seem to have any effect. [The documentation](https://doc.traefik.io/traefik/routing/entrypoints/#redirection), on the other hand, talks about a structure that looks like `entryPoints.web.http.redirections.entryPoint.[to|scheme|permanent|priority]`. Since that's how it's configured everywhere else, why not use the same structure in values.yaml???

We can (and should?) ignore `redirectTo` and just go right to `additionalArguments` instead, using the familiar syntax in the docs:
```yaml
additionalArguments:
  - "--entrypoints.web.http.redirections.entryPoint.to=:443"
  - "--entrypoints.web.http.redirections.entryPoint.scheme=https"
  - "--entrypoints.web.http.redirections.entryPoint.permanent=true"
```

Note the fact that I used `:443` to redirect to, instead of the entryPoint **name** since using `websecure` here will actually cause it to redirect to port 8443. I'm not sure if that's a bug or not in the Helm chart, but this still works and makes sense.

And in case you're not familiar, "permanent" just means the difference between a normal redirect (HTTP 302) and permanent redirect (HTTP 301) which determines which HTTP response is used.

## Other Options
Some of the options you might notice depend on persistent storage. Just be sure you have storage configured in your cluster before using any of those options (which I'll talk about in my next post).

Specifically I noticed the persistence section in the values.yaml file, but that only pertains to storing certificates acquired directly through Traefik. I would just recommend ignoring that option and sticking with cert-manager.

Traefik provides some examples of common use cases which might also be helpful: https://github.com/traefik/traefik-helm-chart/blob/master/EXAMPLES.md

Beyond all of that, if you need to configure custom entryPoints (say you need ingress on a weird port like 4567), read the Traefik docs, make the change in the Helm values and run the upgrade. I might come back later and add a section explaining this in more detail, but for now I just want to mention it's a possibility. I've used it for the GitLab container registry among other things.

## Changed Your Mind After Installing Or Just Need To Upgrade?
This is basic Helm stuff, but basically just update your values and use the same command as before except replacing install with upgrade.
- Update values.yaml
- Upgrade Helm chart: `helm upgrade -n traefik -f values.yaml traefik traefik/traefik`

It almost feels like you just type "traefik traefik traefik" over again a few times, but slightly more nuanced. See the Helm docs for more details.

https://helm.sh/docs/helm/helm_upgrade/

## Traefik Version Upgrade
Always read release notes before upgrading! Sometimes there is more to it than just running the Helm upgrade command. See https://github.com/traefik/traefik-helm-chart?tab=readme-ov-file#upgrading for more details around CRDs and other considerations.

# TLDR; Deploy Using Options I'm Using
- Deployment: `Deployment` with 2 replicas
  - I don't want to run too many replicas with a DaemonSet if I scale up the number of workers.
  - This still helps me if a pod crashes, there is still another replica running while the other pod gets recycled.
- Namespace: `traefik`
  - I'm trying to follow best practices and use namespaces for different things
- Dashboard: Yes
  - I like having access to this in my homelab, but I will protect it with basic auth just as a best practice to secure it in some way.
- HTTP Permanent Redirect: True
  - I don't have any special use cases for HTTP now, but if needed I would just create a new entrypoint on a different port if something needed to be non-TLS.

## Write values.yaml
- `values.yaml` ONLY needs to contain what you're changing, and defaults will be used automatically. So KISS and don't include default values.
  ```yaml
  # Deployment with 2 replicas
  deployment:
    replicas: 2
  
  # Dashboard with basic auth
  ingressRoute:
    dashboard:
      enabled: true
      matchRule: Host(`traefik-dashboard.example.com`)
      entryPoints: ["web","websecure"]
      middlewares:
        - name: traefik-dashboard-auth
  extraObjects:
    - apiVersion: v1
      kind: Secret
      metadata:
        name: traefik-dashboard-auth-secret
      type: kubernetes.io/basic-auth
      stringData:
        # Change the password PLEASE!!!
        username: admin
        password: changeme
    - apiVersion: traefik.io/v1alpha1
      kind: Middleware
      metadata:
        name: traefik-dashboard-auth
      spec:
        basicAuth:
          secret: traefik-dashboard-auth-secret
  
  # HTTP permanent redirect
  additionalArguments:
    - "--entrypoints.web.http.redirections.entryPoint.to=:443"
    - "--entrypoints.web.http.redirections.entryPoint.scheme=https"
    - "--entrypoints.web.http.redirections.entryPoint.permanent=true"
  ```
- Deploy using one-size-fits-all command:
  - `helm upgrade --install -n traefik --create-namespace -f values.yaml traefik traefik/traefik`

# Testing Ingress
Finally, the easy part. Deploy something, then deploy an Ingress resource, and test. Let's try it.

## Deployment
Create a test deployment with 2 replicas. Traefik provides a utility called "whoami" which is helpful. This deployment also includes a Service (defaults to type ClusterIP) which is needed before deploying an Ingress resource.
- Create `whoami-deployment.yaml`:
  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: whoami-deployment
  spec:
    selector:
      matchLabels:
        app: whoami
    replicas: 2
    template:
      metadata:
        labels:
          app: whoami
      spec:
        containers:
        - image: traefik/whoami:latest
          name: whoami
          ports:
          - containerPort: 80
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: whoami-svc
  spec:
    ports:
    - port: 80
      targetPort: 80
      protocol: TCP
    selector:
      app: whoami
  ```
- Create the deployment: `kubectl apply -f whoami-deployment.yaml`
- Verify: `k get deploy`

## Ingress
This is a super basic Ingress resource which can be useful for testing. It includes two annotations:
- One for cert-manager to issue a certificate from the letsencrypt-staging ClusterIssuer for the hostname specified under Ingress.spec.tls.hosts. If you had used an Issuer instead of ClusterIssuer, you would need to change that line to `cert-manager.io/issuer: "letsencrypt-staging"`
- One to specify which entryPoints Traefik should route traffic on. This can be a single entrypoint, or a comma separated list like `web,websecure` - see https://doc.traefik.io/traefik/routing/providers/kubernetes-ingress/#on-ingress

- Create `whoami-ingress.yaml` and update host in both places, and cluster issuer if yours has a different name:
  ```yaml
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: whoami-ingress
    annotations:
      cert-manager.io/cluster-issuer: "letsencrypt-staging"
      traefik.ingress.kubernetes.io/router.entrypoints: websecure
  spec:
    ingressClassName: traefik
    tls:
    - hosts:
      - whoami.example.com
      secretName: whoami-example-tls
    rules:
    - host: whoami.example.com
      http:
        paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: whoami-svc
              port:
                number: 80
  ```
- Verify: `kubectl get ingress`
- Test
  - Update DNS or your local hosts file to point your host (whoami.example.com) to the IP address of Traefik's LoadBalancer service (`kubectl get svc -n traefik`)
  - Navigate to whoami.example.com in the browser. If you test too soon, you will get the self signed cert from Traefik. If you wait long enough and are still using the staging certificate issuer, you will still have to click through the warning message but it should say it's issued by Let's Encrypt Staging. If you use a production issuer, this should be secure with no warnings.

# What About Gateway API?
This Kong article has a lot of good information on the topic: https://konghq.com/blog/engineering/gateway-api-vs-ingress

I have read that Gateway API is the successor to ingress controllers. The official FAQ says that Gateway API will not replace Ingress, so we are safe to continue using them (https://gateway-api.sigs.k8s.io/faq/#will-gateway-api-replace-the-ingress-api). There is no need to use Gateway API if all you need is Ingress.

Gateway API solves some inherent limitations with Ingress, primarily the fact that Ingress is Layer 7 only.

## Gateway API Tutorial?
Nah.

We would need a separate tutorial on Gateway API, so I don't even try to begin here. The examples from above mention that you even have to enable it specifically in the Traefik installation before using it: https://github.com/traefik/traefik-helm-chart/blob/master/EXAMPLES.md#use-kubernetes-gateway-api

Also - How to migrate from Ingress to Gateway API: https://gateway-api.sigs.k8s.io/guides/migrating-from-ingress/