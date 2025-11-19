---
layout: blog
draft: true
title: Kubernetes Homelab Series Part 8 - GitOps With FluxCD
date: 2025-11-19
tags:
  - kubernetes
  - homelab
  - gitops
  - fluxcd
summary: Exploring the basics of GitOps using FluxCD to manage a gitlab-runner deployment.
---
# What is GitOps?
definition go here

# What is FluxCD?
definition go here

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