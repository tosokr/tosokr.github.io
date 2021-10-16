---
title: Building an AKS baseline architecture - Part 3 - GitOps with Flux2
date: 2021-10-16 00:00:00 +0000
description: In this series of posts, you will find all the steps needed to build a baseline or reference architecture for Azure Kubernetes Service (AKS) by incorporating all the best practices from the operations and governance perspective. In this post, in short, we discussed all control plane logging, RBAC role assignments, Azure Container Registries, and Azure Policy.
categories: [Azure Kubernetes Service]
tags: [AKS]
toc: true 
header:
 teaser: "/assets/img/posts/teasers/aks.png"
permalink: /aks/baseline-part-3/
excerpt: In this series of posts, you will find all the steps needed to build a baseline or reference architecture for Azure Kubernetes Service (AKS) by incorporating all the best practices from the operations and governance perspective. In this post, in short, we discussed all control plane logging, RBAC role assignments, Azure Container Registries, and Azure Policy.
---
Posts from this series:

[Building an AKS baseline architecture - Part 1 - Cluster creation]({% post_url 2021-06-26-building-aks-baseline-part-1 %})

[Building an AKS baseline architecture - Part 2 - Governance]({% post_url 2021-07-17-building-aks-baseline-part-2 %})

[Building an AKS baseline architecture - Part 3 - GitOps with Flux2]({% post_url 2021-10-11-building-aks-baseline-part-3 %})

# What is GitOps?

"*GitOps is a way of implementing Continuous Deployment for cloud native applications. It focuses on a developer-centric experience when operating infrastructure, by using tools developers are already familiar with, including Git and Continuous Deployment tools.
The core idea of GitOps is having a Git repository that always contains declarative descriptions of the infrastructure currently desired in the production environment and an automated process to make the production environment match the described state in the repository. If you want to deploy a new application or update an existing one, you only need to update the repository - the automated process handles everything else. It’s like having cruise control for managing your applications in production.*"- [GitOps.tech](https://www.gitops.tech/)

There is a great [GitOps for AKS](https://docs.microsoft.com/en-us/azure/architecture/example-scenario/gitops-aks/gitops-blueprint-aks) document available in the Azure Architecture Center that can help you learn more about the need and the benefits of using the GitOps approach for AKS. In this article, I will focus on the implementation part of GitOps - how to make it work with Flux.

# Flux2

*[Flux](fluxcd.io) is a tool for keeping Kubernetes clusters in sync with sources of configuration (like Git repositories), and automating updates to configuration when there is new code to deploy.*

There are multiple approaches for organizing the git repos when enrolling Flux. In the baseline architecture, we will implement the [monorepo](https://fluxcd.io/docs/guides/repository-structure/#monorepo) approach - a single repository for multiple environments. Environments will have the same base infrastructure components,  but the versions of custom applications will be different:

```
├── apps
│   ├── production 
│   └── staging
├── infrastructure
│   ├── sources
│   ├── base 
└── clusters
    ├── production
    └── staging
```
The design is not optimal for enterprise deployments, but it is a good starting point for learning.

As a best practice, you should accommodate the trank-based development approach: create minor updates in short-living branches that you will merge to main by opening pull requests. For the simplicity of the demo, we will perform all commits directly to our main branch.

## Install Flux2

```bash
export GITHUB_USER=
export GITHUB_TOKEN= # PERSONAL ACCESS TOKEN FOR GITHUB. SCOPE: Full control of private repositories
export GITHUB_REPO=aks-baseline-clusters # REPO FOR FLUX
export GITHUB_PATH=clusters/production # PATH TO USE INSIDE THE REPO 

curl -s https://fluxcd.io/install.sh | sudo bash # INSTALL FLUX LOCALY

# ENABLE COMPLETITIONS in ~/.bash_profile
. <(flux completion bash)

# IMPORT THE FLUX IMAGES IN OUR AZURE CONTAINER REGISTRY (https://aztoso.com/aks/baseline-part-2/#azure-container-registry-acr)
az acr import --source $(flux install --export | grep ghcr.io/fluxcd/helm-controller | awk -F" " '{print $2}') -n $ACR_NAME --force
az acr import --source $(flux install --export | grep ghcr.io/fluxcd/kustomize-controller | awk -F" " '{print $2}') -n $ACR_NAME --force
az acr import --source $(flux install --export | grep ghcr.io/fluxcd/notification-controller | awk -F" " '{print $2}') -n $ACR_NAME --force
az acr import --source $(flux install --export | grep ghcr.io/fluxcd/source-controller | awk -F" " '{print $2}') -n $ACR_NAME --force

# BOOTSTRAP FLUX
flux bootstrap github --owner=$GITHUB_USER --repository=$GITHUB_REPO \
--path=$GITHUB_PATH --personal --registry $ACR_NAME.azurecr.io/fluxcd

# CLONE THE REPO LOCALY
git clone https://$GITHUB_USER:$GITHUB_TOKEN@github.com/$GITHUB_USER/$GITHUB_REPO.git  ../$GITHUB_REPO

```

We will use Azure Container Registry to host our Helm Charts. Because OCI is [not yet supported by Flux](https://fluxcd.io/docs/use-cases/azure/#helm-repositories-on-azure-container-registry), we can't use Managed Identities for authenticating to the ACR. The only option is to use service principal (make sure you regulary rotate the secret):

```bash
# CREATE A SERVICE PRINCIPAL WITH PERMISSION TO PULL FROM THE ACR
fluxHelmAcr=$(az ad sp create-for-rbac -n sp-aks-baseline-$ACR_NAME-acrpull --role "AcrPull" --scope $(az acr show -g $RESOURCE_GROUP_ACR -n $ACR_NAME --query id -o tsv --only-show-errors)
FLUX_HELM_ACR_APPID=$(echo -n $fluxHelmAcr | jq '.appId' -r | tr -d '\n'| base64)
FLUX_HELM_ACR_SECRET=$(echo -n $fluxHelmAcr | jq '.password' -r | tr -d '\n'| base64)

# KUBERNETES SECRET THAT STORES THE SERVICE PRINCIPAL DETAILS
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: flux-helm-acr-credentials
  namespace: flux-system
type: Opaque
data:
  username: ${FLUX_HELM_ACR_APPID}
  password: ${FLUX_HELM_ACR_SECRET}
EOF

# THE KUSTOMIZATION DEFINITION FOR THE CLUSTER:
mkdir -p $GITHUB_REPO/clusters/production
cat > $GITHUB_REPO/clusters/production/infrastructure.yaml << ENDOFFILE
apiVersion: kustomize.toolkit.fluxcd.io/v1beta1
kind: Kustomization
metadata:
  name: infrastructure
  namespace: flux-system
spec:
  interval: 10m0s
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./infrastructure
  prune: true
  validation: client
ENDOFFILE

cd $GITHUB_REPO
git add *
git commit -m "infrastructure production kustomization"
git push
cd ..

# CREATE HELMREPOSITORY MANIFEST FOR THE ACR
mkdir -p $GITHUB_REPO/infrastructure/sources
cat > $GITHUB_REPO/infrastructure/sources/acr.yaml << ENDOFFILE
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: HelmRepository
metadata:
  name: acr-private
  namespace: flux-system
spec:
  url: https://${ACR_NAME}.azurecr.io/helm/v1/repo
  secretRef:
    name: flux-helm-acr-credentials
  interval: 1m
ENDOFFILE

# COMMIT AND PUSH THE CHANGES
cd $GITHUB_REPO
git add *
git commit -m "acr helm repository"
git push
cd ..
```

To verify that  Flux created the Helm repository definition on the cluster, execute the following command:

```bash
kubectl get helmrepositories.source.toolkit.fluxcd.io -n flux-system
```
```
NAME          URL                                                READY   STATUS                                                       AGE
acr-private   https://xw5OUsrKQmU40Im4.azurecr.io/helm/v1/repo   True    Fetched revision: c45818b304c4b776703ab0fc56efbe492690830e   3h9m
```

# Flux2 Helm deployment example: Kured

Kured (Kubernetes Reboot Daemon) is a DaemonSet that triggers a node reboot if a file exists at the predefined path. The underlying OS in AKS, Ubuntu 18.04, performs a security or kernel patch check every night and automatically installs the patches. If the patch requires a restart, AKS will create a/var/run/reboot-required file. In simple words, Kured makes sure that all critical updates are installed on the cluster.

```bash
# PULL THE HELM CHART LOCALY
helm repo add kured https://weaveworks.github.io/kured
helm pull kured/kured

# PUSH THE HELM CHART TO ACR
az acr helm push -n $ACR_NAME kured-*.tgz

# PUSH THE IMAGE TO ACR
KURED_CHART_VERSION=$(helm show chart kured/kured | grep version |  awk -F" " '{print $2}')
az acr import --source docker.io/weaveworks/kured:$(helm show chart kured/kured | grep appVersion |  awk -F" " '{print $2}') \
-n $ACR_NAME --force

# CREATE HELMRELEASE MANIFEST FOR KURED
mkdir -p $GITHUB_REPO/infrastructure/base/kured
cat > $GITHUB_REPO/infrastructure/base/kured/release.yaml << ENDOFFILE
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: kured
spec:
  releaseName: kured
  chart:
    spec:
      chart: kured
      sourceRef:
        kind: HelmRepository
        name: acr-private
        namespace: flux-system
      version: "${KURED_CHART_VERSION}"
  interval: 1h0m0s
  install:
    remediation:
      retries: 3
  # Default values
  # https://github.com/weaveworks/kured/blob/main/charts/kured/values.yaml
  values:
    tolerations:
    - key: CriticalAddonsOnly
      operator: Exists
    image:
      repository: ${ACR_NAME}.azurecr.io/weaveworks/kured
ENDOFFILE

# NAMESPACE TO USE FOR KURED
cat > $GITHUB_REPO/infrastructure/base/kured/namespace.yaml << ENDOFFILE
apiVersion: v1
kind: Namespace
metadata:
  name: baseline
ENDOFFILE

# KUSTOMIZATION MANIFEST FOR KURED
cat > $GITHUB_REPO/infrastructure/base/kured/kustomization.yaml << ENDOFFILE
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: baseline
resources:
  - namespace.yaml
  - release.yaml
ENDOFFILE

# COMMIT AND PUSH THE CHANGES
cd $GITHUB_REPO
git add *
git commit -m "kured installation"
git push
cd ..
```