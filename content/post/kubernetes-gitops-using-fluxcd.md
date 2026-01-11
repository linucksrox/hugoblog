---
layout: blog
draft: true
title: Kubernetes Homelab Series Part 8 - GitOps With FluxCD
date: 2026-01-11
tags:
  - kubernetes
  - homelab
  - gitops
  - fluxcd
summary: Exploring FluxCD to manage deployments and releases using GitOps.
---
# What is GitOps?
[GitLab defines GitOps](https://about.gitlab.com/topics/gitops/) as "GitOps is an operational framework that takes DevOps best practices used for application development such as version control, collaboration, compliance, and CI/CD, and applies them to infrastructure automation."

In my own words, I would explain it as a way to manage deployments by defining desired state in a git repo, then automatically keeping deployments in sync with the repo by running a tool that is constantly monitoring the difference between desired state and actual state, and automatically reconciling when needed.

This is the next iteration of IaC (Infrastructure as Code) which already got us to store our desired application state in a git repo. GitOps expands on IaC by automatically monitoring and reconciling desired state (your git repo) with running state (what's running in the cluster).

This provides additional benefits as well, including:
- Simpler deployments - You no longer need to manually deploy after updating your desired state, or write custom deployment pipelines
- Simpler security - Rather than pushing code from your repo, often requiring networking and allow listing, GitOps flips this to a pull-based deployment where the controller runs within the cluster, watching your repo and reconciling changes as needed.

# What is FluxCD?
FluxCD is one of several GitOps tools for Kubernetes. Flux is a great into to GitOps because it's simple to get started, and it's a CNCF Graduated project so it's stable and works very well, not to mention having a lot of users and great community in case you get stuck or have questions.

# Walkthrough
- Install flux CLI
- Generate a PAT and export
  - `export GITLAB_TOKEN=glpat-...`
- Bootstrap flux
  - Update `hostname` with your instance hostname, `owner` with your group name, `repository` with the name you want to use for the repo, `branch` with the name of the main/master branch you want to use, and `path` with the directory structure within the repo you want to use.
  ```
  flux bootstrap gitlab \
  --deploy-token-auth \
  --hostname=gitlab.example.com \
  --owner=gitops \
  --repository=fluxcd-demo \
  --branch=master \
  --path=clusters/my-cluster
  ```
- something