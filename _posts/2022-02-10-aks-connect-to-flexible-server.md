---
title: Private Link and Flexible Server DNS resolution using custom VNET resolvers
date: 2022-02-10 00:00:00 +0000
description: In enterprise scenarios, when you need to resolve on-premise DNS records or have cross-subscription DNS resolution of the private DNS zones, configuring proper DNS resolution for Private (Link) Endpoints and Flexible Server resources can be challenging. In a nutshell, for Private Link resources, use single DNS zones hosted in a central subscription. In contrast, use custom DNS resolvers for Flexible Servers in each subscription or apply custom CoreDNS configuration for resolution from AKS clusters. 
categories: [Azure Kubernetes Service,Networking]
tags: [AKS,FlexibleServer]
toc: true 
header:
 teaser: "/assets/img/posts/teasers/privateDNSZones.png"
permalink: /networking/dns-private-link-flexible-server/
excerpt: In enterprise scenarios, when you need to resolve on-premise DNS records or have cross-subscription DNS resolution of the private DNS zones, configuring proper DNS resolution for Private (Link) Endpoints and Flexible Server resources can be challenging. In a nutshell, for Private Link resources, use single DNS zones hosted in a central subscription. In contrast, use custom DNS resolvers for Flexible Servers in each subscription or apply custom CoreDNS configuration for resolution from AKS clusters. 
---

# Overview

In enterprise scenarios, when you need to resolve on-premise DNS records or have cross-subscription DNS resolution of the private DNS zones, you need to have some kind of hybrid DNS configuration. One such setup is having the DNS resolvers in a central subscription with all Private DNS zones linked to the VNET those resolvers reside in. 

# Private (Link) Endpoints

If you plan to create many private endpoints across your subscriptions, the above approach will not scale. A better solution is to have a single private DNS zone per service linked to the resolvers VNET and add the records there during the endpoint creation. To make it work, after you create the private endpoint for the service in your VNET you need to create a [DNS Zone Group](https://docs.microsoft.com/en-us/azure/private-link/private-endpoint-dns#private-dns-zone-group):

```bash
az network private-endpoint dns-zone-group create 
    --endpoint-name # the name of the enpoint you created
    --name # name of the dns-zone-group (free to use whatever you like)
    --private-dns-zone # resource id of the private DNS zone
    --resource-group # resource group where you created the private endpoint
    --zone-name # name of the private DNS zone
    [--subscription] # if endpoint is created in another subscription
# example
az network private-endpoint dns-zone-group create 
    --endpoint-name privatelink-storage \
    --resource-group rg-privatelink-test \
    --name test-dns-zone-group \
    --private-dns-zone "/subscriptions/<SUBSCRIPTION_ID>/resourceGroups/rg-test-privatelink/providers/Microsoft.Network/privateDnsZones/privatelink.blob.core.windows.net" \
    --zone-name privatelink.blob.core.windows.net
```
To be able to create that DNS Zone Group, you need the following permissions on the private DNS zone:

```bash
"Microsoft.Network/privateDnsZones/A/read",
"Microsoft.Network/privateDnsZones/A/write",
"Microsoft.Network/privateDnsZones/read",
"Microsoft.Network/privateDnsZones/join/action",
```
To create the Private DNS Zone Groups through the portal, you also need the _Microsoft.Resources/subscriptions/resourceGroups/read_ permission on the resource group hosting the Private DNS Zone.

# Flexible Server

Flexible Server uses VNET integration to provide private access to the databases. Under the hood, when you create the service, you delegate a dedicated subnet to Microsoft to deploy and operate the VMs used for hosting the database server for you. As part of the VNET integration, the deployment creates a Private DNS zone with the name equal to the server's name, making it impossible to use the same solution as with the private endpoints. The only possibility to overcome this challenge is to host custom DNS resolvers in the (spoke) subscriptions:

![Desktop View]({{ "/assets/img/posts/networking/vnet-custom-resolvers.png" | relative_url }})

## DNS resolution from AKS

Suppose you only need to connect to the Flexible Server from AKS. In that case, instead of deploying custom DNS resolvers in the VNET, you can [customize CoreDNS](https://docs.microsoft.com/en-us/azure/aks/coredns-custom) to forward all the queries for Flexible Server to the Azure DNS Server. For PostgreSQL, the custom configuration will look like this:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns-custom
  namespace: kube-system
data:
  flexible.server: |
   postgres.database.azure.com:53 {
      errors
      cache 30
      forward . 168.63.129.16
      reload
    }
EOF
```

After you apply the coredns-custom configmap, force CoreDNS to reload the ConfigMap (as per the documentation, the command bellow will not cause any downtime):

```bash
kubectl delete pod --namespace kube-system -l k8s-app=kube-dns
```

