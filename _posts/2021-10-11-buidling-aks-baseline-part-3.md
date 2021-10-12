---
title: Building an AKS baseline architecture - Part 3 - GitOps
date: 2021-06-27 00:00:00 +0000
description: In this series of posts, you will find all the steps needed to build a baseline or reference architecture for Azure Kubernetes Service (AKS) by incorporating all the best practices from the operations and governance perspective. In this post, in short, we discussed all control plane logging, RBAC role assignments, Azure Container Registries, and Azure Policy.
categories: [Azure Kubernetes Service]
tags: [AKS]
toc: true 
header:
 teaser: "/assets/img/posts/teasers/aks.png"
permalink: /aks/baseline-part-2/
excerpt: In this series of posts, you will find all the steps needed to build a baseline or reference architecture for Azure Kubernetes Service (AKS) by incorporating all the best practices from the operations and governance perspective. In this post, in short, we discussed all control plane logging, RBAC role assignments, Azure Container Registries, and Azure Policy.
---

"*GitOps is a way of implementing Continuous Deployment for cloud native applications. It focuses on a developer-centric experience when operating infrastructure, by using tools developers are already familiar with, including Git and Continuous Deployment tools.
The core idea of GitOps is having a Git repository that always contains declarative descriptions of the infrastructure currently desired in the production environment and an automated process to make the production environment match the described state in the repository. If you want to deploy a new application or update an existing one, you only need to update the repository - the automated process handles everything else. It’s like having cruise control for managing your applications in production.*"- [GitOps.tech](https://www.gitops.tech/)

There is a great [GitOps for AKS](https://docs.microsoft.com/en-us/azure/architecture/example-scenario/gitops-aks/gitops-blueprint-aks) document available in the Azure Architecture Center that can help you learn more about the need and the benefits of using the GitOps apprach for AKS.

# Flux2

*[Flux](fluxcd.io) is a tool for keeping Kubernetes clusters in sync with sources of configuration (like Git repositories), and automating updates to configuration when there is new code to deploy*

We will use the [monorepo](https://fluxcd.io/docs/guides/repository-structure/#monorepo) approach - single repository for multiple environments. Using the trank-based development approach, we will create small updates in a short-living branches that we will merge it to main by opening pull requests. 

```bash

# LIST ALL CONTAINER IMAGES FLUX NEEDS
flux install --export | grep ghcr.io

# IMPORT THE FLUX IMAGES IN AZURE CONTAINER REGISTRY
az acr import --source ghcr.io/fluxd/helm-controller:v0.12.0 -n $ACR_NAME --force
az acr import --source ghcr.io/fluxcd/kustomize-controller:v0.15.4 -n $ACR_NAME --force
az acr import --source ghcr.io/fluxcd/notification-controller:v0.17.0 -n $ACR_NAME --force
az acr import --source ghcr.io/fluxcd/source-controller:v0.16.0 -n $ACR_NAME --force

# INSTALL FLUX
export GITHUB_USER=
export GITHUB_TOKEN=
export GITHUB_REPO=aks-baseline-clusters
export GITHUB_PATH=clusters/production

curl -s https://fluxcd.io/install.sh | sudo bash

# ENABLE COMPLETITIONS in ~/.bash_profile
. <(flux completion bash)

# BOOTSTRAP FLUX
flux bootstrap github --owner=$GITHUB_USER --repository=$GITHUB_REPO \
--path=$GITHUB_PATH --personal --registry $ACR_NAME.azurecr.io/fluxcd

# CLONE THE REPO LOCALY
git clone https://$GITHUB_USER:$GITHUB_TOKEN@github.com/$GITHUB_USER/$GITHUB_REPO.git  ../$GITHUB_REPO

# CREATE NEW WORKING BRANCH
cd ../$GITHUB_REPO
git checkout -b feature/initial

```