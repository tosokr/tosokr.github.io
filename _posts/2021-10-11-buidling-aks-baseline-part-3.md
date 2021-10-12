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
