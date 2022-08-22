---
title: Deep dive into Azure Container Apps - Overview and Networking
date: 2022-08-22 00:00:00 +0000
description: Azure Container Apps is a serverless offering you can use to host your containers. It is a good fit for containerized apps and hosting microservices. Integrated services like KEDA, Envoy proxy, and Dapr provide you with out-of-the-box auto-scaling, ingress, traffic splitting, and simplified microservice connectivity. Container Apps service is built on top of Kubernetes.
categories: [Container Apps]
tags: [Kubernetes, ContainenerApps]
toc: true
header:
 teaser: "/assets/img/posts/teasers/container_apps.jpg"
permalink: /container-apps/deep-dive-overview-and-networking/
excerpt: Azure Container Apps is a serverless offering you can use to host your containers. It is a good fit for containerized apps and hosting microservices. Integrated services like KEDA, Envoy proxy, and Dapr provide you with out-of-the-box auto-scaling, ingress, traffic splitting, and simplified microservice connectivity. Container Apps service is built on top of Kubernetes.
---

# Overview
[Azure Container Apps](https://docs.microsoft.com/en-us/azure/container-apps/overview) is a serverless offering you can use to host your containers. It is a good fit for containerized apps and hosting microservices. Integrated services like KEDA, Envoy proxy, and Dapr provide you with out-of-the-box auto-scaling, ingress, traffic splitting, and simplified microservice connectivity.

Container Apps service is built on top of Kubernetes. Container Apps are an Azure Resource Manager deployment object, meaning you can't just use your existing Kubernetes object descriptions and migrate them to Container Apps. You need to rewrite your deployment stack using Bicep or ARM templates. Terraform is not yet supported.

For each Container App, you need to specify the resources you want to allocate. The current range starts from 0.25 vCPU and 0.5Gi memory up to 2 vCPUs and 4.0Gi memory. If your container needs more resources to operate, you must brake your business logic down into multiple containers (or refer to them as microservices). Container Apps is not a good fit for you if that is not possible.

This document provides deep dive into Container Apps by clarifying not well-documented features, provides recommendations for deployments and gives examples of how to perform some of the actions. For a complete reference, check the [official product page](https://docs.microsoft.com/en-us/azure/container-apps/).

## When are Container Apps a good fit?
If you want to avoid the complexity of operating a Kubernetes cluster, Container Apps are a good fit. It means that you need to trade off the control you have in Kubernetes for the simplicity the Container Apps service brings. For example, with Kubernetes, you are free to add platform features like automatic certificate rotations, external DNS registration, etc. In Container Apps, you need to wait for Microsoft to introduce such features in the platform. 

If you want to host a website, App Service is a much better choice for that. If the website is a microservice application composed of multiple containers, Container Apps is a better fit.

Azure Container Instances is the preferred option for running single isolated containers that perform background processing and don't need to scale out.

## Dapr
[Dapr (Distributed Application Runtime)](https://dapr.io/) is a portable, serverless, event-driven runtime that makes it easy for developers to build resilient, stateless, and stateful microservices that run on the cloud and edge and embraces the diversity of languages and developer frameworks. Dapr building blocks expose common distributed application capabilities, such as state management, service-to-service invocation, and pub/sub messaging

If you enable Dapr in Container Apps, it will inject a sidecar container which you can use for the microservices connectivity. You enable Dapr per Container App, meaning that in a Container Apps Environment, you can have apps with and without Dapr support. After the sidecar injection, you can talk to the Dapr container via HTTP or gRPC. Calls between Dapr sidecars are always made with gRPC, which delivers high performance. Dapr is cloud-agnostic and provides language-specific SDKs for easy development. The benefit of Dapr is that it speeds up application development by eliminating the need for a plumbing code while simultaneously, built applications are platform (cloud) agnostic. 

# Networking
## Ingress controller
An Envoy proxy is used on the backend to provide ingress connectivity to the Container Apps. As part of the Container Apps deployment, you can configure the accessibility level (public, VNET, or Container Apps Environment), traffic split rules (when using multiple revisions), custom DNS name, and certificate to associate with the ingress. Other configuration options for the ingress are not available. Because the ingress is part of the platform, you don't need to take care of updating it.

By default, ports 80 and 443 are open with HTTP to HTTPS redirection. As a general security guideline, ALWAYS enable only HTTPS traffic, even when you allow connection only from Container Apps Environment. 

The fully qualified domain name (FQDN) associated with the ingress depends on the accessibility level:

| Ingress accessibility | FQDN |
|-------|--------|
| Public | 	APP_NAME.UNIQUE_IDENTIFIER.REGION_NAME.azurecontainerapps.io |
| VNET only | APP_NAME.UNIQUE_IDENTIFIER.REGION_NAME.azurecontainerapps.io |
| Container Apps Environment only | APP_NAME.internal.UNIQUE_IDENTIFIER.REGION_NAME.azurecontainerapps.io |

**Note:** mTLS is not (yet) supported by the ingress controller.  Use OAuth2 to secure your application.

## Custom DNS
Azure Container Apps allows you to bind one or more custom domains to a container app. For each custom domain, you also need a Server Name Indication (SNI) domain certificate associated with the ingress.

Steps to follow:

* Upload a private key certificate (.pfx or .pem) to the Container Apps Environment.
* In Container Apps, link the certificate with the custom DNS domain name

There is no possibility of storing the certificate in a Key Vault. By developing an automation script, you can rotate the certificate on a scheduled basis.

## Communication between Container Apps
If you enable ingress,  your Container App will be exposed using a dedicated DNS name. You can use that name to reach the Container App from other Container Apps. It is worth mentioning that the communication between Container Apps deployed in the same environment never leaves the environment:

![Desktop View]({{ "/assets/img/posts/container-apps/communication.png" | relative_url }})

As you can see from the picture, the environment DNS suffix is available inside containers via the environment variable CONTAINER_APP_ENV_DNS_SUFFIX.  You can append that value to the name of the Container App you want to establish a connection to. 

If you enable Dapr, you can levarage the built-in service discovery (the command is executed from another container app, using the console):

![Desktop View]({{ "/assets/img/posts/container-apps/communication-dapr.png" | relative_url }})

## VNET integration
By default, when you create Container Apps Environment, it uses a managed VNET, that is a VNET that is part of the service, and you can't see it or perform any configurational changes. If you want to:

* use private endpoints (Private Link) to connect to your service (for example, database )
* establish a private connection to a service that offers a VNET integration (such as Flexible Server for PostgreSQL) 
* communicate to a service that is available only on your internal network

then you need to use a custom VNET when creating the Container Apps Environment.

With custom VNET, you need a minimum of /23 subnet dedicated to Container Apps Environment. When deployed, the service reserves 60 IPs per VMSS configuration, meaning that if you enable Availability Zone, it will reserve 3 x 60 = 180 IPs.

### Custom VNET with an internal environment
When using an internal environment (without a public exposure), the underneath Kubernetes cluster is exposed using a Private Load Balancer, which consumes a single IP address from the subnet where it is deployed. In the managed resource group (starts with mc_) there is also a Public Load Balancer deployed, which is used only for outbound connections, providing Internet connectivity to underlying Kubernetes nodes and deployed revisions.

To resolve the DNS names of your Container Apps from the VNET, you need to:

1. Create a Private DNS Zone using the Container Apps Environment default domain name. The default domain is in the format: VARIABLE_NAME.LOCATION.azurecontainerapps.io. For example icyplant-362dc571.westeurope.azurecontainerapps.io
2. Create a "star" DNS record inside the zone, pointing to the Internal Load Balancer IP address.
3. Link the Private DNS Zone with your VNET

![Desktop View]({{ "/assets/img/posts/container-apps/internal_evironment_with_custom_vnet.png" | relative_url }})

It is impossible to use a custom private DNS zone for the Container Apps, because the service can't verify the ownership. You have two options:

* use a public DNS zone to create a private IP record
* create a link to container apps environment Private DNS zone to all VNETs that need access to your service

If you choose to enable the HTTP ingress, you can:

* limit the traffic to the VNET - everyone who has network connectivity to your VNET will potentially have access 
* limit to the Container Apps Environment - the app will be reachable only within the environment by other Container Apps

### Custom VNET with an external environment
With an external environment, the configuration is more straightforward because there is no need for a private DNS zone integration into the VNET.

![Desktop View]({{ "/assets/img/posts/container-apps/external_evironment_with_custom_vnet.png" | relative_url }})

If you choose to enable the HTTP ingress, you can:

* limit to the Container Apps Environment - the app will be reachable only within the environment by other Container Apps
* accept traffic from everywhere - everyone who has network connectivity to your VNET will potentially have access 

Suppose you want to apply an allow list to the incoming connections. In that case, you must create a Network Security Group (NSG) and associate it to the subnet where the Container Apps are deployed. The rule should look like this:

* Source: List of source IP addresses
* Source port ranges: * (usually allow all source ports)
* Destination: The public IP address of the environment. ![Desktop View]({{ "/assets/img/posts/container-apps/environment_ip.png" | relative_url }})
* Destination port ranges: 443

**Note:** Never modify the NSG (aks-agentpool-{id}-nsg) that is deployed in the mc_ managed resource group. The Container Apps service controls that NSG, and it can be overwritten.

