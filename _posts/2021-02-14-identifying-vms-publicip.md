---
title: Identifying VMs with Public IP address
date: 2021-02-14 00:00:00 +0000
description: Identifying all the VMs that are reachable from the Internet is something you must do to govern your environment successfully. It is not a simple task to perform that operation because your environment can consist of multiple subscriptions organized in a management group hierarchy. When you need to query against Azure, Resource Graph is your best friend. It provides efficient and performant resource exploration with the ability to query at scale across a given set of subscriptions so that you can effectively govern your environment.
categories: [Virtual Machines]
tags: [Resource Graph]
toc: false 
header:
 teaser: "/assets/img/posts/teasers/resourceGraphQueries.png"
permalink: /vm/identifying-vms-publicip/
excerpt: Identifying all the VMs that are reachable from the Internet is something you must do to govern your environment successfully. It is not a simple task to perform that operation because your environment can consist of multiple subscriptions organized in a management group hierarchy. When you need to query against Azure, Resource Graph is your best friend. It provides efficient and performant resource exploration with the ability to query at scale across a given set of subscriptions so that you can effectively govern your environment.
---
Identifying all the VMs that are reachable from the Internet is something you must do to govern your environment successfully. It is not a simple task to perform that operation because your environment can consist of multiple subscriptions organized in a management group hierarchy. When you need to query against Azure, [Resource Graph](https://azure.microsoft.com/en-us/features/resource-graph/) is your best friend. It provides efficient and performant resource exploration with the ability to query at scale across a given set of subscriptions so that you can effectively govern your environment.

To identifies all the VMs that have Public IP attached to them: 
```
Resources
| where type =~ 'Microsoft.Compute/virtualMachines'
| project subscriptionId, resourceGroup, name, networkInterfaces = (properties.networkProfile.networkInterfaces)
| mv-expand networkInterfaces
| project subscriptionId, resourceGroup, vmName = name, networkInterfaceId = tostring(networkInterfaces.id)
| join kind=leftouter(
    Resources
    | where type =~ 'Microsoft.Network/networkInterfaces'
    | project id, ipConfigurations = (properties.ipConfigurations)
    | mv-expand ipConfigurations
    | project id, publicIpAddressId = tostring(ipConfigurations.properties.publicIPAddress.id) 
    | join kind = leftouter (
        Resources
        | where type =~ 'Microsoft.Network/publicIPAddresses'
        | project publicIpId=id, ipAddress=tostring(properties.ipAddress)
    ) on $left.publicIpAddressId == $right.publicIpId
) on  $left.networkInterfaceId == $right.id
| where notempty(ipAddress)
| project subscriptionId, resourceGroup, vmName, ipAddress
```

To identify the Virtual Machine Scale Sets with a Public Load balancer:
```
Resources
| where type =~ 'Microsoft.Compute/virtualMachineScaleSets'
| project subscriptionId, resourceGroup, name, networkInterfaceConfigurations = (properties.virtualMachineProfile.networkProfile.networkInterfaceConfigurations)
| mv-expand networkInterfaceConfigurations
| project subscriptionId, resourceGroup, name, ipConfigurations = (networkInterfaceConfigurations.properties.ipConfigurations)
| mv-expand ipConfigurations
| project subscriptionId, resourceGroup, name, loadBalancerBackendAddressPools = (ipConfigurations.properties.loadBalancerBackendAddressPools)
| mv-expand loadBalancerBackendAddressPools
| where isnotempty(loadBalancerBackendAddressPools.id)
| project subscriptionId, resourceGroup, name, loadBalancerBackendAddressPoolsId = (loadBalancerBackendAddressPools.id), loadBalancerId=substring(loadBalancerBackendAddressPools.id,0, indexof(loadBalancerBackendAddressPools.id, '/backendAddressPools/'))
| join kind = leftouter(
    Resources
    |  where type =~ 'Microsoft.Network/loadBalancers'
    | project id, loadBalancerProperties = properties
) on $left.loadBalancerId == $right.id
| project subscriptionId, resourceGroup, name, frontendIPConfigurations=(loadBalancerProperties.frontendIPConfigurations)
| mv-expand frontendIPConfigurations
| project subscriptionId, resourceGroup, name, publicIPId=tostring(frontendIPConfigurations.properties.publicIPAddress.id)
| join kind = leftouter (
    Resources
    |  where type =~ 'Microsoft.Network/publicIPAddresses'
    | project id,  ipAddress=tostring(properties.ipAddress)
) on $left.publicIPId == $right.id
| distinct subscriptionId, resourceGroup, name, ipAddress
```