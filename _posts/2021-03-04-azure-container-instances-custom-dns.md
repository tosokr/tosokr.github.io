---
title: Azure Container Instances in VNET with custom DNS
date: 2021-03-04 00:00:00 +0000
description: Azure Container Instances (ACI) is a serverless container runtime offering. You can use it to deploy Linux containers into an Azure virtual network, which will make them reachable from your networks. If you are using custom DNS resolvers in the VNET where you plan to deploy your ACI, the containers will not inherit that configuration. By default, they will use the Azure DNS service with the virtual IP address of 168.63.129.16.
categories: [Azure Container Instances]
tags: [ACI]
toc: false 
header:
 teaser: "/assets/img/posts/teasers/aci.png"
permalink: /aci/azure-container-instances-custom-dns/
excerpt: Azure Container Instances (ACI) is a serverless container runtime offering. You can use it to deploy Linux containers into an Azure virtual network, which will make them reachable from your networks. If you are using custom DNS resolvers in the VNET where you plan to deploy your ACI, the containers will not inherit that configuration. By default, they will use the Azure DNS service with the virtual IP address of 168.63.129.16.
---
Azure Container Instances (ACI) enables you to run containers on-demand without managing or thinking about the infrastructure below. In summary, it is a serverless container runtime offering. It supports both Linux and Windows containers. 

You can deploy Linux containers into an Azure virtual network, which will make them reachable from your networks. If you are using custom DNS resolvers in the VNET where you plan to deploy your ACI, the containers will not inherit that configuration. By default, they will use the Azure DNS service with the virtual IP address of 168.63.129.16.

To specify your custom DNS servers, you need to use YAML file for the deployments. The following is an example of the content of such a file:
```yaml
apiVersion: '2021-07-01'
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
      - <YOUR DNS SERVER>
      - <YOUR DNS SERVER>
  subnetIds: # Subnet to deploy the container group into
    - id: <RESOURCE ID OF YOUR SUBNET - in a format /subscriptions/<subscription_id>/resourceGroups/<resource_group_name>/providers/Microsoft.Network/virtualNetworks/<virtual_network_name>/subnets/<subnet_name>>
      name: <SUBNET NAME>
  ipAddress:
    type: Private
    ports:
    - protocol: tcp
      port: '80'
  osType: Linux  
type: Microsoft.ContainerInstance/containerGroups
```
You then create a Container Instance using the yaml file:

```bash
az container create -g <RESOURCE GROUP NAME> -f containerGroup.yml
```

To verify the DNS configuration, connect to the console of your container, and check the content of the _/etc/resolv.conf_ file:

```bash
cat /etc/resolv.conf 
```