---
title: Building an AKS baseline architecture - Part 2
date: 2021-06-27 00:00:00 +0000
description: In this series of posts, you will find all the steps needed to build a baseline or reference architecture for Azure Kubernetes Service (AKS) by incorporating all the best practices from the operations and governance perspective. In this post, in short, we discussed all the base AKS components, and we deployed a base AKS cluster. 
categories: [Azure Kubernetes Service]
tags: [AKS]
toc: true 
header:
 teaser: "/assets/img/posts/teasers/aks.png"
permalink: /aks/baseline-part-1/
excerpt: In this series of posts, you will find all the steps needed to build a baseline or reference architecture for Azure Kubernetes Service (AKS) by incorporating all the best practices from the operations and governance perspective. In this post, in short, we discussed all the base AKS components, and we deployed a base AKS cluster. 
---

In the first part of this blog series, we created the base AKS cluster following the best practices. Now it is time to apply some governance controls on top of it.

# Control plane logging
By default, logging is not enabled for the AKS's control plane components. It would be best to store the control plane logs to audit the operations against the cluster and for troubleshooting purposes. We can enable the logs by [configuring diagnostics settings for the AKS cluster](https://docs.microsoft.com/en-us/azure/aks/view-control-plane-logs).

For our baseline configuration, we are going to enable the following log categories:
- _kube-audit-admin_ - contains all audit log data for every audit event, excluding get and list.
- _kube-controller-manager_ - internal cluster operations, such as replicating pods
- _cluster-autoscaler_ - cluster scaling operations
- _guard_ - managed Azure AD and Azure RBAC audits

In addition to that, we will save all of the metrics into a dedicated log analytics workspace.

To apply the diagnostics settings using a CLI, we need to create two JSON files, one for the log categories and one for the metrics:

```bash
cat << 'EOF' > aksLogCategories.json
[
    {
        "category": "kube-apiserver",
        "enabled": false,
        "retentionPolicy": {
            "enabled": false,
            "days": 0
        }
    },
    {
        "category": "kube-audit",
        "enabled": false,
        "retentionPolicy": {
            "enabled": false,
            "days": 0
        }
    },
    {
        "category": "kube-audit-admin",
        "enabled": true,
        "retentionPolicy": {
            "enabled": true,
            "days": 30
        }
    },
    {
        "category": "kube-controller-manager",
        "enabled": true,
        "retentionPolicy": {
            "enabled": true,
            "days": 30
        }
    },
    {
        "category": "kube-scheduler",
        "enabled": false,
        "retentionPolicy": {
            "enabled": false,
            "days": 0
        }
    },
    {
        "category": "cluster-autoscaler",
        "enabled": true,
        "retentionPolicy": {
            "enabled": true,
            "days": 30
        }
    },
    {
        "category": "guard",
        "enabled": true,
        "retentionPolicy": {
            "enabled": true,
            "days": 30
        }
    }
]
EOF

cat << 'EOF' > aksMetrics.json
[
    {
        "category": "AllMetrics",
        "enabled": true,
        "retentionPolicy": {
            "enabled": true,
            "days": 30
        }
    }
]
EOF
```

Next, we will create the log analytics workspace for storing that data in a dedicated resource group: 
```bash
RESOURCE_GROUP_AUDIT=$RESOURCE_GROUP-audit
LOGANALYTICS_NAME_AUDIT=$LOGANALYTICS_NAME-audit

resourceGroupExists=$(az group exists --name "$RESOURCE_GROUP_AUDIT")
if [ "$resourceGroupExists" == "false" ]; then 
    echo "Creating resource group: "$RESOURCE_GROUP_AUDIT" in location: ""$LOCATION"
    az group create --name "$RESOURCE_GROUP_AUDIT" --location "$LOCATION"
fi

logAnalyticsExists=$(az monitor log-analytics workspace list --resource-group $RESOURCE_GROUP_AUDIT --query "[?name=='$LOGANALYTICS_NAME_AUDIT'].name" -o tsv)
if [ "$logAnalyticsExists" != "$LOGANALYTICS_NAME_AUDIT" ]; then
    az monitor log-analytics workspace create --resource-group $RESOURCE_GROUP_AUDIT \
    --workspace-name $LOGANALYTICS_NAME_AUDIT --location $LOCATION
fi
```

Finally, we will enable the diagnostics settings for our cluster: 
```bash
az monitor diagnostic-settings create --resource "$AKS_RESOURCE_ID" --name "AksLogging" --workspace "$LOGANALYTICS_NAME_AUDIT" \
    --resource-group $RESOURCE_GROUP_AUDIT --logs '@aksLogCategories.json' --metrics '@aksMetrics.json'
```
# Azure Role Based Access Control (RBAC) role assignments
When using [Azure RBAC for Kubernetes Authorization](https://docs.microsoft.com/en-us/azure/aks/manage-azure-rbac) Azure AD principals are validated by Azure RBAC while regular Kubernetes users and service accounts are validated by Kubernetes RBAC. 

Four built-in roles exists for the RBAC assignments inside the cluster:
- _Azure Kubernetes Service RBAC Reader_ - Allows read-only access to see most objects in a namespace, excluding secrets, roles and role bindings. You can apply this role on a particular namespace or at the cluster scope.
- _Azure Kubernetes Service RBAC Writer_ - Allows read/write access to most objects in a namespace, excluding roles and role bindings. You can apply this role on a particular namespace or at the cluster scope.
- _Azure Kubernetes Service RBAC Admin_ - Lets you manage all resources, except update or delete resource quotas and namespaces. You can apply this role on a particular namespace or at the cluster scope.
- _Azure Kubernetes Service RBAC Cluster Admin_ - Lets you manage all resources in the cluster.	Applicable only at the cluster scope.

Don't mess this roles with the RBAC roles used to control actions against the cluster itself:
- _Azure Kubernetes Service Cluster Admin Role_ - List cluster admin credential action
- _Azure Kubernetes Service Cluster User Role_ - List cluster user credential action
- _Azure Kubernetes Service Contributor Role_ - Grants access to read and write Azure Kubernetes Service clusters

As a best practice, always assign the RBAC roles using an Azure AD group. If you have Privilaged Identity Management (PIM) enabled on your tenant, use an escalation procedure to get membership to the group that has assigned the _Azure Kubernetes Service RBAC Cluster Admin_ role. Another best practice is to use the role id when creating the assignments, because the role names can change over time.

```bash
# ASSIGN THE CLUSTER ADMIN RBAC ROLE (id: b1ff04bb-8a4e-4dc4-8eb5-8693973ce19b) TO AN AZURE AD GROUP
AZURE_AD_GROUP_NAME=""
AZURE_AD_GROUP_OBJECT_ID=$(az ad group show -g RG_AZ_CCoE_Azure --query objectId -o tsv) 
az role assignment create --role b1ff04bb-8a4e-4dc4-8eb5-8693973ce19b --assignee-object-id $AZURE_AD_GROUP_OBJECT_ID --assignee-principal-type Group --scope $AKS_RESOURCE_ID
```
New role assignments can take up to five minutes to propagate and be updated by the authorization server.

On top of this, to be able to login to the cluster, you need to have one of the two types of cluster credentials:
- _User credentials_ - allows you to login to the cluster. Your cluster role depends from the RBAC assignments. You need to have at least the _Azure Kubernetes Service Cluster User Role_ assigned on the AKS resource. _Azure Kubernetes Service Contributor Role_, _Contributor_ or _Owner_ will also work. 
- _Admin credentials_ - allows you to login to the cluster as cluster admin. You don't need to have any RBAC assingments. Use the admin credentials only for break-glass scenarios, for example, when you are not able to login using Azure AD due to some system failures. You need to have  the _Azure Kubernetes Service Cluster Admin Role_ assigned on the AKS resource. _Contributor_ or _Owner_ will also work. Available in preview, you can [disable the local admin credentials](https://docs.microsoft.com/en-us/azure/aks/managed-aad#disable-local-accounts-preview) and force all authentication to happen via Azure AD.

To get the cluster credentials, use the following command:
```bash
# USER CREDENTIALS
az aks get-credentials -g $RESOURCE_GROUP -n $CLUSTER_NAME

# ADMIN CREDENTIALS (DON'T USE IT FOR DAILY OPERATIONS)
az aks get-credentials -g $RESOURCE_GROUP -n $CLUSTER_NAME --admin
```
Install [kubectx](https://github.com/ahmetb/kubectx) to be able to swich between the different kubernetes clusters or even between the user and admin credentials for the same cluster. 

# Azure Container Registry (ACR)

[Azure Container Registry](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-intro) is a private registry service for building, storing, and managing container images and related artifacts. As a best practice, for your production workloads, don't rely on the public registries for hosting your application's or dependencies container images. Instead, deploy your own registy over which you have a full control and you can implement governance controls.

ACR offers [three tiers](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-skus): Basic, Standard and Premium. The Premium SKU offers geo-replication and availability zones support (in preview) and it reccomended for enterprise deployments.

You can [authenticate to the ACR](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-authentication?tabs=azure-cli) using Azure AD (with user or service credentials) or with an admin user. Best practice is to disable the admin user and don't use that in production. AKS cluster will authenticate to the ACR using its managed identity.

```bash
# CREATE THE CONTAINER REGISTRY
ACR_NAME=$(cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 16 | head -n 1) # NEEDS TO BE UNIQUE
RESOURCE_GROUP_ACR=$RESOURCE_GROUP-acr

resourceGroupExists=$(az group exists --name "$RESOURCE_GROUP_ACR")
if [ "$resourceGroupExists" == "false" ]; then 
    echo "Creating resource group: "$RESOURCE_GROUP_ACR" in location: ""$LOCATION"
    az group create --name "$RESOURCE_GROUP_ACR" --location "$LOCATION"
fi

acrExists=$(az acr list -g $RESOURCE_GROUP_ACR --query "[?name=='$ACR_NAME'].name" -o tsv)
if [ "$acrExists" != "$ACR_NAME" ]; then
   ACR_ID=$(az acr create -g $RESOURCE_GROUP_ACR -n $ACR_NAME --sku Basic --admin-enabled false --location $LOCATION --query id -o tsv)
else
   ACR_ID=$(az acr list -g $RESOURCE_GROUP_ACR --query "[?name=='$ACR_NAME'].id" -o tsv ) 
fi
```
Next, we need to give permissions to the AKS cluster to be able to pull images from the ACR.
```bash
# ALLOW ACCESS TO THE ACR FROM AKS
az aks update -n $CLUSTER_NAME -g $RESOURCE_GROUP --attach-acr $ACR_NAME
```
The previous command assings the AcrPull role to the Kubelet Identity (User Managed Identity assigned to the nodepools). To verify this:
```bash
CLUSTER_IDENTITY_ID=$(az aks show -g $RESOURCE_GROUP -n $CLUSTER_NAME --query identityProfile.kubeletidentity.clientId -o tsv)
az role assignment list --assignee $CLUSTER_IDENTITY_ID --scope $ACR_ID
```  
## Azure Defender for Container Registries

# Azure Policies-
