---
title: Building an AKS baseline architecture - Part 1
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
## Overview
Azure Kubernetes Service (AKS) is a managed Kubernetes cluster offering by Microsoft. Everything around AKS is pretty well documented in the [official documentation](https://docs.microsoft.com/en-us/azure/aks/). The idea of this post series is not to copy/paste what is already well documented out there but to put in place everything you need to build a baseline deployment for AKS, following the best practices. 
Then, to learn more about the particular features of that baseline deployment, links are provided in the appropriate section.
We will be using the imperative way of deploying by leveraging the Azure CLI because it is easier to view the actual workflow fully, and it is much easier to learn by going step-by-step. However, it is strongly recommended to use a declarative tool for production workloads, such as Terraform or Bicep.  

## AKS Cluster
### AKS Cluster components
#### Network Plugin
You can deploy AKS cluster using two [network plugins](https://docs.microsoft.com/en-us/azure/aks/concepts-network) out-of-the-box: Kubenet or Azure CNI. We will use Azure CNI because it is a prerequisite for using [Virtual Nodes](https://docs.microsoft.com/en-us/azure/aks/virtual-nodes). Another reason why to use Azure CNI instead of Kubenet is because Kubenet network plugin is susceptible to ARP spoofing which is a real issue when using [Pod Identities](https://azure.github.io/aad-pod-identity/docs/configure/aad_pod_identity_on_kubenet/). Last, Azure CNI is considered first class citizen in AKS because usually all the new features in AKS are at first developed only for CNI.

#### Network Policy
The Network Policy feature in Kubernetes enables you to define rules for ingress and egress traffic between pods in a cluster. The Calico network policy supports [more features](https://docs.microsoft.com/en-us/azure/aks/use-network-policies#overview-of-network-policy), including support for both Azure CNI and Kubenet network plugins, and it is the default and recommended network policy solution to use.

#### Cluster Identity
AKS needs its own [identity](https://docs.microsoft.com/en-us/azure/aks/use-managed-identity) to create additional resources like load balancers and managed disks.It is recommend to use System Assigned Identity because the lifetime of it is bind to the lifetime of the AKS cluster and credentials (certificates) are automatically rotated by Azure every 46 days. Another options to use User Assigned Identity or manually create Service Principal (strongly not recommended!).

#### Azure RBAC
The new [Azure RBAC for Kubernetes Authorization](https://docs.microsoft.com/en-us/azure/aks/manage-azure-rbac) feature will enable us to manage RBAC for Kubernetes resources from Azure. When enabled, Azure AD principals will be validated exclusively by Azure RBAC while regular Kubernetes users and service accounts are exclusively validated by Kubernetes RBAC. To make it work, we will also enable the [Managed Azure AD Integration](https://docs.microsoft.com/en-us/azure/aks/manage-azure-rbac)

#### Kubernetes version
By default, the Azure CLI deploys the latest stable version on Kubernetes. You can specify a custom version, but keep in mind to satisfy the N-2 (N (Latest release) - 2 (minor versions)) [constrain](https://docs.microsoft.com/en-us/azure/aks/supported-kubernetes-versions). If you fail to do so, you risk running a non-supported cluster not covered by the AKS’s SLO or SLA, which you should not do for your production workloads.

#### Service Level Agreement(SLA) for AKS
The [uptime SLA](https://docs.microsoft.com/en-us/azure/aks/uptime-sla) is an optional feature to get financially backed SLA of 99.95% for the Kubernetes API server endpoint for clusters that use Availability Zones and 99.9% of availability for clusters that don’t use Availability Zones. The price is ~€50. If you don’t enable the uptime SLA because the AKS control plane (API Server) is a free service offered by Azure, you will get an Service Level Objective(SLO) of 99.5%. As a best practice, always enable this feature for production deployments.

For the node pools, the SLA of the VMs applies. If you deploy the node pool in a VMSS distributed across Availability Zones and have a minimum of two nodes up and running, you will get a 99.99% SLA. If at some point, your node pool consists only of one instance, then the [SLA of a single VM](https://azure.microsoft.com/en-us/support/legal/sla/virtual-machines) applies. 

#### Node pools
Two types of [node pools](https://docs.microsoft.com/en-us/azure/aks/use-multiple-node-pools) exists in AKS:
- System node pool - hosts critical system pods that are mandatory for the cluster to operate. Such type of pod is, for example, CoreDNS.
- User node pool - hosts your application pods
If you don’t configure any [taints](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/), the system node pool will be able to host also application pods. The same applies to the user node pool. By setting the CriticalAddonsOnly=true:NoSchedule taint, we will use a dedicated system node pool.
A minimum of two nodes, each with 4 vCPU are [reccomended](https://docs.microsoft.com/en-us/azure/aks/use-system-pools#system-and-user-node-pools) for the system node pool. Best practice is to enable the autoscaller, and set the maximum number of nodes to 5.
For the user nodes, it mainly depends from the application's resource needs. Change the configuration as needed.

##### Max-surge
The max-surge settings on the nodepool determine the number of nodes that can be drained simultaneously when performing an AKS upgrade. The provided value of 33% is a best practice to use for production clusters.

##### Virtual Machine SKU
Standard_D4ds_v4 SKU is used for the system node pool. This SKU provides 4 vCPU with 16GB of RAM. For production workloads, always make sure you choose an SKU with a minimum of 4 vCPUs. Use the latest VM SKU. You will get better performances, usually for the same or lower price. Check the available [VM sizes](https://docs.microsoft.com/en-us/azure/virtual-machines/sizes), and the features they support. Pay attention to the CPU:Memory ratio, Azure Compute Units (ACU), and temporary/cache disk space support (important for ephemeral disks).

##### Ephemeral disks
[Ephemeral OS disk](https://docs.microsoft.com/en-us/azure/virtual-machines/ephemeral-os-disks) enables lower read/write latency since the disk is locally attached. This also leads to faster cluster operations like scale or upgrade. The usage of Ephemeral disks depends on the chosen VM SKU for the node pool. With ephemeral OS you can deploy VM and instance images up to the size of the VM cache. In the AKS case, the default node OS disk configuration uses 128GB, which means that you need a VM size with a cache larger than 128GB. Another option is to reduce the size of the OS disk to fit into the available cache. The minimum size for AKS images is 30GB. The Standard_D4ds_v4 SKU supports 100GB cache size, and we will use that cache size to provision and Ephemeral disk.

##### Host-based encryption
To encrypt the Ephemeral OS disk at rest, we will enable the [host-based encryption](https://docs.microsoft.com/en-us/azure/aks/enable-host-encryption)  

##### Availability Zones
AKS clusters deployed in multiple [availability zones](https://docs.microsoft.com/en-us/azure/aks/availability-zones) provide a higher level of availability. This pattern helps to protect against a hardware failures, or a planned maintenance event. The usage of Availability Zones is configurable per node pool. Usage of Availability Zones also increases the SLA (of VMSS) to 99.99%, in case two or more nodes are up and running.
In the baseline, we are going to use three (the maximum number of) availability zones per node pool. If you have a latency sensitive workload, you may consider to use a single zone for the user node pool hosting that workload. 
Volumes that use Azure managed disks are currently not zone-redundant resources (the feature is available in [preview](https://docs.microsoft.com/en-us/azure/virtual-machines/disks-redundancy?tabs=azure-cli#zone-redundant-storage-for-managed-disks-preview)). Volumes cannot be attached across zones and must be co-located in the same zone as a given node hosting the target pod.

### Deployment prerequisites
As part of our prerequisites for deploying the cluster, we need to create a resource group, virtual network with a subnet, and log analytics workspace for storing Kubernetes logs and container insights. We also need to register several providers and features in our subscription for the features we will use.
```bash
# VARIABLES
SUBSCRIPTION_ID=""
RESOURCE_GROUP="rg-aks-baseline"
LOCATION="westeurope"
CLUSTER_NAME="aks-baseline"
VNET_NAME="vnet-aks-baseline"
VNET_ADDRESS_SPACE="10.10.0.0/16"
VNET_SUBNET_NAME="snet-"$CLUSTER_NAME
VNET_SUBNET_ADDRESS_SPACE="10.10.0.0/22"
ACI_SUBNET_NAME="snet-aci-"$CLUSTER_NAME
ACI_SUBNET_ADDRESS_SPACE="10.10.4.0/23"
LOGANALYTICS_NAME="log-"$CLUSTER_NAME
LOGANALYTICS_RETENTION_DAYS=30 #30-730

# LOGIN TO THE SUBSCRIPTION
az login 
az account set --subscription $SUBSCRIPTION_ID

# REGISTER THE AZURE POLICY PROVIDER
az provider register --namespace Microsoft.PolicyInsights

# REGISTER PROVIDERS FOR CONTAINER INSIGHTS
az provider register --namespace Microsoft.OperationsManagement
az provider register --namespace Microsoft.OperationalInsights

# REGISTER THE ENCRYPTION-AT-HOST FEATURE
az feature register --namespace Microsoft.Compute --name EncryptionAtHost

# CREATE THE RESOURCE GROUP
resourceGroupExists=$(az group exists --name "$RESOURCE_GROUP")
if [ "$resourceGroupExists" == "false" ]; then 
    echo "Creating resource group: "$RESOURCE_GROUP" in location: ""$LOCATION"
    az group create --name "$RESOURCE_GROUP" --location "$LOCATION"
fi

# CREATE THE VNET
vnetExists=$(az network vnet list -g "$RESOURCE_GROUP" --query "[?name=='$VNET_NAME'].name" -o tsv)
if [ "$vnetExists" != "$VNET_NAME" ]; then
    az network vnet create --resource-group $RESOURCE_GROUP --name $VNET_NAME \
    --address-prefix $VNET_ADDRESS_SPACE
fi

# CREATE SUBNET FOR THE CLUSTER 
subnetExists=$(az network vnet subnet list -g "$RESOURCE_GROUP" --vnet-name $VNET_NAME --query "[?name=='$VNET_SUBNET_NAME'].name" -o tsv)
if [ "$subnetExists" != "$VNET_SUBNET_NAME" ]; then
    VNET_SUBNET_ID=$(az network vnet subnet create --resource-group $RESOURCE_GROUP --name $VNET_SUBNET_NAME \
    --address-prefixes $VNET_SUBNET_ADDRESS_SPACE --vnet-name $VNET_NAME --query id -o tsv)
else
    VNET_SUBNET_ID=$(az network vnet subnet list -g "$RESOURCE_GROUP" --vnet-name $VNET_NAME --query "[?name=='$VNET_SUBNET_NAME'].id" -o tsv)
fi

# CREATE SUBNET FOR AZURE CONTAINER INSTANCES (ACI)
subnetExists=$(az network vnet subnet list -g "$RESOURCE_GROUP" --vnet-name $VNET_NAME --query "[?name=='$ACI_SUBNET_NAME'].name" -o tsv)
if [ "$subnetExists" != "$ACI_SUBNET_NAME" ]; then
    ACI_SUBNET_ID=$(az network vnet subnet create --resource-group $RESOURCE_GROUP --name $ACI_SUBNET_NAME \
    --address-prefixes $ACI_SUBNET_ADDRESS_SPACE --vnet-name $VNET_NAME --query id -o tsv)
else
    ACI_SUBNET_ID==$(az network vnet subnet list -g "$RESOURCE_GROUP" --vnet-name $VNET_NAME --query "[?name=='$ACI_SUBNET_NAME'].id" -o tsv)
fi


# CREATE LOG ANALYTICS WORKSPACE
logAnalyticsExists=$(az monitor log-analytics workspace list --resource-group $RESOURCE_GROUP --query "[?name=='$LOGANALYTICS_NAME'].name" -o tsv)
if [ "$logAnalyticsExists" != "$LOGANALYTICS_NAME" ]; then
    az monitor log-analytics workspace create --resource-group $RESOURCE_GROUP \
    --workspace-name $LOGANALYTICS_NAME --location $LOCATION --retention-time $LOGANALYTICS_RETENTION_DAYS
fi

```
### Create the AKS Cluster

The *az aks create* command doesn't support setting the node-taints on the default node pool it makes during cluster creation. For that reason, we will first create a "temp" node pool, and afterward, we will replace it with a proper dedicated system node pool.

```bash
# VARIABLES
SYSTEM_NODE_VM_SIZE="Standard_D4ds_v4"
SYSTEM_NODE_OS_DISK_SIZE=100

# CREATE THE CLUSTER
aksClusterExists=$(az aks list -g $RESOURCE_GROUP --query "[?name=='$CLUSTER_NAME'].name" -o tsv)
if [ "$aksClusterExists" != "$CLUSTER_NAME" ]; then
    AKS_RESOURCE_ID=$(az aks create -g $RESOURCE_GROUP -n $CLUSTER_NAME \
    --generate-ssh-keys --location $LOCATION --node-vm-size $SYSTEM_NODE_VM_SIZE --nodepool-name systemtemp --node-count 1 \
    --node-osdisk-type Ephemeral --node-osdisk-size $SYSTEM_NODE_OS_DISK_SIZE --zones {1,2,3} \
    --network-policy calico --network-plugin azure --vnet-subnet-id $VNET_SUBNET_ID  --aci-subnet-name $ACI_SUBNET_NAME \
    --enable-managed-identity --enable-aad --enable-azure-rbac --enable-addons monitoring,azure-policy,virtual-node \
    --workspace-resource-id "/subscriptions/$SUBSCRIPTION_ID/resourcegroups/$RESOURCE_GROUP/providers/microsoft.operationalinsights/workspaces/$LOGANALYTICS_NAME" \
    --yes --query id -o tsv --only-show-errors )  
else
    AKS_RESOURCE_ID==$(az aks show -g $RESOURCE_GROUP -n $CLUSTER_NAME --query id -o tsv --only-show-errors)
fi
```
Next, we will create a dedicated system node pool and delete the one previously created as part of the cluster deployment.
```bash
SYSTEM_NODE_NAME="system"
SYSTEM_MIN_COUNT=2
SYSTEM_MAX_COUNT=5

aksSystemNodePoolExists=$(az aks nodepool list -g $RESOURCE_GROUP --cluster-name $CLUSTER_NAME --query "[?name=='$SYSTEM_NODE_NAME'].name" -o tsv --only-show-errors)
if [ "$aksSystemNodePoolExists" != "$SYSTEM_NODE_NAME" ]; then
    az aks nodepool add -g $RESOURCE_GROUP --cluster-name $CLUSTER_NAME \
    --name $SYSTEM_NODE_NAME --node-vm-size $SYSTEM_NODE_VM_SIZE --enable-cluster-autoscaler \
    --node-osdisk-type Ephemeral --node-osdisk-size $SYSTEM_NODE_OS_DISK_SIZE --zones {1,2,3} \
    --max-count $SYSTEM_MAX_COUNT --min-count $SYSTEM_MIN_COUNT --mode System \
    --vnet-subnet-id $VNET_SUBNET_ID --max-surge 33% --node-taints CriticalAddonsOnly=true:NoSchedule 
    # delete the existing "temp" system  node pool
    az aks nodepool delete -g $RESOURCE_GROUP --cluster-name $CLUSTER_NAME -n systemtemp
fi
```
We will also add a dedication user node pool for hosting our application pods
```bash
USER_NODE_NAME="user1"
USER_NODE_VM_SIZE="Standard_D4ds_v4"
USER_NODE_OS_DISK_SIZE=100
USER_MIN_COUNT=1
USER_MAX_COUNT=4

aksUserNodePoolExists=$(az aks nodepool list -g $RESOURCE_GROUP --cluster-name $CLUSTER_NAME --query "[?name=='$USER_NODE_NAME'].name" -o tsv --only-show-errors)
if [ "$aksUserNodePoolExists" != "$USER_NODE_NAME" ]; then
    az aks nodepool add -g $RESOURCE_GROUP --cluster-name $CLUSTER_NAME \
    --node-osdisk-type Ephemeral --node-osdisk-size $USER_NODE_OS_DISK_SIZE --zones {1,2,3} \
    --name $USER_NODE_NAME --node-vm-size $USER_NODE_VM_SIZE --enable-cluster-autoscaler \
    --max-count $USER_MAX_COUNT --min-count $USER_MIN_COUNT --mode User \
    --vnet-subnet-id $VNET_SUBNET_ID --max-surge 33%
fi
```

## Next
In the next part, we will configure logging for the AKS control plane, create appropriate RBAC roles, deploy Azure Container Registry (ACR) and Azure Defender for Container Registries and deploy some policies to govern our cluster.