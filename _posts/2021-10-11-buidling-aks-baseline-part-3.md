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

We will use the [monorepo](https://fluxcd.io/docs/guides/repository-structure/#monorepo) approach - single repository for multiple environments. As a best practice, use the trank-based development approach, where you will create small updates in a short-living branches that you will merge it to main by opening pull requests. For a simplicity of the demo, we will do all updates on the master.

```bash
# IMPORT THE FLUX IMAGES IN AZURE CONTAINER REGISTRY
az acr import --source $(flux install --export | grep ghcr.io/fluxcd/helm-controller | awk -F" " '{print $2}') -n $ACR_NAME --force
az acr import --source $(flux install --export | grep ghcr.io/fluxcd/kustomize-controller | awk -F" " '{print $2}') -n $ACR_NAME --force
az acr import --source $(flux install --export | grep ghcr.io/fluxcd/notification-controller | awk -F" " '{print $2}') -n $ACR_NAME --force
az acr import --source $(flux install --export | grep ghcr.io/fluxcd/source-controller | awk -F" " '{print $2}') -n $ACR_NAME --force

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

```
We will use Azure Container Registry to host our Helm Charts. Because OCI is [not yet supported by Flux](https://fluxcd.io/docs/use-cases/azure/#helm-repositories-on-azure-container-registry), we can't use Managed Identities for authenticating to the ACR. The only option is to use service principal (make sure you regulary rotate the secret):

```bash
fluxHelmAcr=$(az ad sp create-for-rbac -n sp-aks-baseline-$ACR_NAME-acrpull --role "AcrPull" --scope $(az acr show -g $RESOURCE_GROUP_ACR -n $ACR_NAME --query id -o tsv) --only-show-errors)
FLUX_HELM_ACR_APPID=$(echo -n $fluxHelmAcr | jq '.appId' -r | base64)
FLUX_HELM_ACR_SECRET=$(echo -n $fluxHelmAcr | jq '.password' -r | base64)

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
```

Add the ACR as a source Helm repository to the cluster:

```bash
mkdir -p $GITHUB_REPO/infrastructure/sources
cat > $GITHUB_REPO/infrastructure/sources/acr.yaml << ENDOFFILE
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: HelmRepository
metadata:
  name: acr-${ACR_NAME}
  namespace: flux-system
spec:
  url: https://${ACR_NAME}.azurecr.io/helm/v1/repo
  secretRef:
    name: flux-helm-acr-credentials
  interval: 1m
ENDOFFILE
```

```bash
cd $GITHUB_REPO
git add *
git commit -m "acr helm repository"
git push
cd ..
```

# Kured - Kubernetes Reboot Daemon

Kured is a daemonset that triggers a node reboot if file named /var/run/reboot-required is present on the node. AKS can create that file if it install a security or kernel patch that requires reboot. The underlaying OS in AKS, Ubuntu 18.04, performs that patch check every night. 

```bash
# PULL THE HELM CHART LOCALY
helm pull kured/kured
# PUSH THE HELM CHART TO ACR
az acr helm push -n $ACR_NAME kured-*.tgz
# PUSH THE IMAGE TO ACR
KURED_CHART_VERSION=$(helm show chart kured/kured | grep version |  awk -F" " '{print $2}')
az acr import --source docker.io/weaveworks/kured:$(helm show chart kured/kured | grep appVersion |  awk -F" " '{print $2}') -n $ACR_NAME --force

mkdir -p $GITHUB_REPO/infrastructure/kured

cat > $GITHUB_REPO/infrastructure/kured/release.yaml << ENDOFFILE
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
        name: acr-${ACR_NAME}
        namespace: flux-system
      version: "${KURED_CHART_VERSION}"
  interval: 1h0m0s
  install:
    remediation:
      retries: 3
  # Default values
  # https://github.com/weaveworks/kured/blob/main/charts/kured/values.yaml
  values:
    image:
      repository: ${ACR_NAME}.azurecr.io/weaveworks/kured
ENDOFFILE

cat > $GITHUB_REPO/infrastructure/kured/namespace.yaml << ENDOFFILE
apiVersion: v1
kind: Namespace
metadata:
  name: baseline
ENDOFFILE

cat > $GITHUB_REPO/infrastructure/kured/kustomization.yaml << ENDOFFILE
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: baseline
resources:
  - namespace.yaml
  - release.yaml
ENDOFFILE
```
```bash
cd $GITHUB_REPO
git add *
git commit -m "kured installation"
git push
cd ..
```