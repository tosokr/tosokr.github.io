---
title: Building an AKS baseline architecture - Part 4 - AAD Pod Identity
date: 2022-01-04 00:00:00 +0000
description: In this series of posts, you will find all the steps needed to build a baseline or reference architecture for Azure Kubernetes Service (AKS) by incorporating all the best practices from the operations and governance perspective. In this post, we will explore one of the core baseline componets - AAD Pod Identity. 
categories: [Azure Kubernetes Service]
tags: [AKS]
toc: true 
header:
 teaser: "/assets/img/posts/teasers/aks.png"
permalink: /aks/baseline-part-4/
excerpt: In this series of posts, you will find all the steps needed to build a baseline or reference architecture for Azure Kubernetes Service (AKS) by incorporating all the best practices from the operations and governance perspective. In this post, we will explore one of the core baseline componets - AAD Pod Identity. 
---
Posts from this series:

[Building an AKS baseline architecture - Part 1 - Cluster creation]({% post_url 2021-06-26-building-aks-baseline-part-1 %})

[Building an AKS baseline architecture - Part 2 - Governance]({% post_url 2021-07-17-building-aks-baseline-part-2 %})

[Building an AKS baseline architecture - Part 3 - GitOps with Flux2]({% post_url 2021-10-11-building-aks-baseline-part-3 %})

[Building an AKS baseline architecture - Part 4 - AAD Pod Identity]({% post_url 2022-01-04-building-aks-baseline-part-4 %})

# Overview

When we develop solutions, they usually include multiple Azure services that need to communicate between them. Moving the compute part of the solution to AKS leaves the same problem - who are we going to authenticate the communication between those Azure services? With the traditional approach, using service principals or connection keys, we have the problem of safely storing and rotating the credentials. Microsoft solved this problem by introducing Managed Identities. We can use Managed Identities for the [AKS Control plane and Kubelet](https://docs.microsoft.com/en-us/azure/aks/use-managed-identity), but can we use them for pods inside AKS? The answer is YES - with the open-source component named [AAD Pod Identity]((https://github.com/Azure/aad-pod-identity).

AAD Pod Identity enables Kubernetes applications to access cloud resources securely with Azure Active Directory. In addition to the security benefit, Managed Identities uses long-lived tokens and can handle Azure AD hiccups between 12 and  24 hours.

Note:  In preview, a managed add-on is available that you can enable for your AKS cluster, which will provide you with the same functionality. However, this add-on will never transition to General Availability because Microsoft is working on a V2 version.


# How it works?

The solution consists of two core components:

- Managed Identity Controller (MIC) - manage the AzureAssignedIdentity. Assigns the specified Users Managed Identity to the nodepool (VMSS) on a pod scheduling event.
- Node Managed Identity (NMI) - intercept the requests to the Azure Instance Metadata Service and get the correct token based on the AzureIdentityBindings

and four Custom Resource Definitions (CRDs):

- AzureIdentity - describes the Azure Identity resource. It supports: User Assigned Identity and Service Principal (with password or certificate)
- AzureIdentityBindings - enables binding of the AzureIdentity to a pod using the specified selector
- AzureAssignedIdentity -  managed by the Managed Identity Controller, it describes the state of the identity binding
- AzurePodIdentityException - enables pods to access the Azure Instance Metadata Service directly, without the request being intercepted by NMI

The following diagram describe the complete workflow of how AAD Pod Identity works:

![Desktop View]({{ "/assets/img/posts/aks/aad-pod-identity.png" | relative_url }})


# Role Based Access Control Assignments

The AKS cluster needs to have the necessary permissions to assign and unassign the user-managed identities on the underlying Virtual Machine Scale Set. You need to have the role of Owner or User Access Administrator to perform RBAC assignments in Azure.

To create the assignments using Azure CLI, use the instructions below. Consider automating the process and do the assignments using a pipeline:

```bash
RESOURCE_GROUP= # resource group where the AKS cluster is deployed
CLUSTER_NAME=  # the name of the AKS cluster
SUBSCRIPTION_ID= #  the id of the subscription
IDENTITIES_RESOURCE_GROUP= # resource group name where the identities you plan to assign to pods are stored. If you plan to bind their lifetime to the cluster, you can keep them in the NODE_RESOURCE_GROUP. Otherwise, it is a best practice to use a dedicated resource group

# Obtain the ID of the managed identity assigned on the node pools
CLUSTER_IDENTITY_ID=$(az aks show -g $RESOURCE_GROUP -n $CLUSTER_NAME --query identityProfile.kubeletidentity.clientId -o tsv)

# Get the node resource group
NODE_RESOURCE_GROUP=$(az aks show -g $RESOURCE_GROUP -n $CLUSTER_NAME --query nodeResourceGroup -o tsv --only-show-errors)

# Assign the necessary role to the AKS cluster identity, to be able to modify the VMSS settings
az role assignment create --role "Virtual Machine Contributor" --assignee $CLUSTER_IDENTITY_ID --scope /subscriptions/$SUBSCRIPTION_ID/resourcegroups/$NODE_RESOURCE_GROUP

# Option 1 - Assign the necessary role to the AKS cluster identity, to be able to attach the managed identities in the IDENTITIES_RESOURCE_GROUP to the VMSS.
az role assignment create --role "Managed Identity Operator" --assignee $CLUSTER_IDENTITY_ID --scope /subscriptions/$SUBSCRIPTION_ID/resourcegroups/$IDENTITIES_RESOURCE_GROUP

# Option 2 - Assing the necessary role to the AKS cluster identity only to particular user managed identities:
az role assignment create --role "Managed Identity Operator" --assignee $CLUSTER_IDENTITY_ID --scope /subscriptions/$SUBSCRIPTION_ID/resourcegroups/$IDENTITIES_RESOURCE_GROUP/providers/Microsoft.ManagedIdentity/userAssignedIdentities/$IDENTITY_NAME
```

# Security considerations with Kubenet

Running aad-pod-identity in a cluster with Kubenet network plugin is not a recommended configuration because of the security implication. Kubenet is susceptible to ARP spoofing, making it possible for other pods to impersonate a pod with access to identity. Using CAP_NET_RAW capability, an attacker pod could impersonate a pod with valid access and then request a token.

To mitigate the vulnerability, you need to add a securityContext in your pod defintion that drops the NET_RAW capability:

```yaml
securityContext:
  capabilities:
    drop:
    - NET_RAW
```

To enforce this mitigation at the cluster level, use [Azure Policy add-on for AKS](https://aztoso.com/aks/baseline-part-2/#azure-policies) and enroll the [_Kubernetes cluster containers should only use allowed capabilities_](https://portal.azure.com/#blade/Microsoft_Azure_Policy/PolicyDetailBlade/definitionId/%2Fproviders%2FMicrosoft.Authorization%2FpolicyDefinitions%2Fc26596ff-4d70-4e6a-9a30-c2506bd2f80c) policy. Add NET_RAW into _Required drop capabilities_ in the policy parameter values.

# Deploy with Flux2

```bash
# PULL THE HELM CHART LOCALY
helm repo add aad-pod-identity https://raw.githubusercontent.com/Azure/aad-pod-identity/master/charts
helm pull aad-pod-identity/aad-pod-identity

# PUSH THE HELM CHART TO ACR
az acr helm push -n $ACR_NAME aad-pod-identity-*.tgz

# PUSH THE IMAGE TO ACR
AADPODIDENTITY_CHART_VERSION=$(helm show chart aad-pod-identity/aad-pod-identity | grep version |  awk -F" " '{print $2}')
az acr import --source mcr.microsoft.com/oss/azure/aad-pod-identity/nmi:v$(helm show chart aad-pod-identity/aad-pod-identity | grep appVersion |  awk -F" " '{print $2}') \
-n $ACR_NAME --force
az acr import --source mcr.microsoft.com/oss/azure/aad-pod-identity/mic:v$(helm show chart aad-pod-identity/aad-pod-identity | grep appVersion |  awk -F" " '{print $2}') \
-n $ACR_NAME --force

# CREATE HELMRELEASE MANIFEST FOR AAD-POD-IDENTITY
mkdir -p $GITHUB_REPO/infrastructure/base/aad-pod-identity
cat > $GITHUB_REPO/infrastructure/base/aad-pod-identity/release.yaml << ENDOFFILE
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: aad-pod-identity
spec:
  releaseName: aad-pod-identity
  chart:
    spec:
      chart: aad-pod-identity
      sourceRef:
        kind: HelmRepository
        name: acr-private
        namespace: flux-system
      version: "${AADPODIDENTITY_CHART_VERSION}"
  interval: 1h0m0s
  install:
    remediation:
      retries: 3
  # Default values
  # https://github.com/Azure/aad-pod-identity/blob/master/charts/aad-pod-identity/values.yaml
  values:    
    image:
      repository: ${ACR_NAME}.azurecr.io/oss/azure/aad-pod-identity
ENDOFFILE

# KUSTOMIZATION MANIFEST FOR AAD-POD-IDENTITY
cat > $GITHUB_REPO/infrastructure/base/aad-pod-identity/kustomization.yaml << ENDOFFILE
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: baseline
resources:  
  - release.yaml
ENDOFFILE

# COMMIT AND PUSH THE CHANGES
cd $GITHUB_REPO
git add *
git commit -m "aad-pod-identity installation"
git push
cd ..

```

