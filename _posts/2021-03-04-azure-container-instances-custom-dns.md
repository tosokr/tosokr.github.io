---
title: Azure Container Instances in VNET with custom DNS
date: 2021-03-04 00:00:00 +0000
description: Azure Container Instances (ACI) is a serverless container runtime offering. You can use it to deploy Linux containers into an Azure virtual network, which will make them reachable from your networks. If you are using custom DNS resolvers in the VNET where you plan to deploy your ACI, the containers will not inherit that configuration. By default, they will use the Azure DNS service with the virtual IP address of 168.63.129.16.
categories: [Azure Container Instances]
tags: [ACI]
toc: false 
header:
 teaser: "/assets/img/posts/teasers/aci.png"
permalink: /logic-app/keyvault-connector-with-managed-identity/
excerpt: Azure Container Instances (ACI) is a serverless container runtime offering. You can use it to deploy Linux containers into an Azure virtual network, which will make them reachable from your networks. If you are using custom DNS resolvers in the VNET where you plan to deploy your ACI, the containers will not inherit that configuration. By default, they will use the Azure DNS service with the virtual IP address of 168.63.129.16.
---
Azure Container Instances (ACI) enables you to run containers on-demand without managing or thinking about the infrastructure below. In summary, it is a serverless container runtime offering. It supports both Linux and Windows containers. 

You can deploy Linux containers into an Azure virtual network, which will make them reachable from your networks. If you are using custom DNS resolvers in the VNET where you plan to deploy your ACI, the containers will not inherit that configuration. By default, they will use the Azure DNS service with the virtual IP address of 168.63.129.16.

To specify your custom DNS servers, you need to use YAML file for the deployments. The following is an example of the content of such a file:
```yaml
apiVersion: '2019-12-01'
location: westeurope
name: customdns
properties:
  containers:
  - name: alpine01
    properties:
      image: alpine
      command:
      - tail
      - -f
      - /dev/null
      resources:
        requests:
          cpu: 1
          memoryInGb: 1.5
      ports:
      - port: 80
  dnsConfig:
    nameServers:
      - 10.10.10.10
        10.10.10.11
  ipAddress:
    type: Private
    ports:
    - protocol: tcp
      port: '80'
  osType: Linux
  networkProfile:
    id: /subscriptions/<subscription_id>/resourceGroups/<resource_group>/providers/Microsoft.Network/networkProfiles/<name_of the_profile>
type: Microsoft.ContainerInstance/containerGroups
```
When you execute the "az container create" command to deploy a container group to a subnet, a [network profile](https://docs.microsoft.com/en-us/azure/container-instances/container-instances-virtual-network-concepts#network-profile) is created for you. To find out the id of your network profile, use the following command:
```bash
az network profile list --resource-group <resource_group_name> \
  --query [0].id --output tsv
```