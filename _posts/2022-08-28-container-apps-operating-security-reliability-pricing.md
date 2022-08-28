---
title: Deep dive into Azure Container Apps - Operating, security, reliability and pricing
date: 2022-08-22 00:00:00 +0000
description: Azure Container Apps is a serverless offering you can use to host your containers. It is a good fit for containerized apps and hosting microservices. Integrated services like KEDA, Envoy proxy, and Dapr provide you with out-of-the-box auto-scaling, ingress, traffic splitting, and simplified microservice connectivity. Container Apps service is built on top of Kubernetes.
categories: [Container Apps]
tags: [Kubernetes, ContainenerApps]
toc: true
header:
 teaser: "/assets/img/posts/teasers/container_apps.jpg"
permalink: /container-apps/deep-dive-operating-security-reliability-pricing/
excerpt: Azure Container Apps is a serverless offering you can use to host your containers. It is a good fit for containerized apps and hosting microservices. Integrated services like KEDA, Envoy proxy, and Dapr provide you with out-of-the-box auto-scaling, ingress, traffic splitting, and simplified microservice connectivity. Container Apps service is built on top of Kubernetes.
---
[Deep dive into Azure Container Apps - Overview and Networking]({% post_url 2022-08-22-container-apps-overview-and-networking %})

# Operating Container Apps
## Revisions
In short, revisions are the versions of the Container App. A new revision is created when a revision-scope change is applied to the Container App, such as a change in the configuration or an image version. A Container App can have single or multiple active revisions at the same time. Multiple revisions enable traffic splitting, where a predefined percentage of the traffic can hit particular revisions, which is helpful in A/B testing and Blue/Green deployments scenarios.

By default, the revision name is composed by applying an alphanumerical random suffix to the Container App Name, like CONTAINER_APP_NAME--RANDOM_SUFFIX.  It is recommended to replace the random with a custom suffix by using the commit id of your deployment repo (populate the value dynamically during the deployment).The commit id is helping in the case when you need to apply revision-scope changes without changing the image version. The release tag is not a good candidate because it usually contains dots that are not a good accepted in the revision suffix.

The [revision labels](https://docs.microsoft.com/en-us/azure/container-apps/revisions#revision-labels) can help you directly reach the desired revision if you plan to use traffic splitting. This can be useful if you want to perform some smoke test before gradually switching the traffic to the latest version (revision) or perform a full switch when using blue/green deployments. With that in mind, having the "latest" revision label applied to the last revision makes sense. You can move the label between the revisions as desired.

```
traffic: [         
    {           
        revisionName: '${containerAppName}--${previousRevisionSuffix}'
        weight: 100
    }
    {
        revisionName: '${containerAppName}--${newRevisionSuffix}'
        label: 'latest'
        weight: 0
    }
]
```

**Note:** When using the multiple revision mode, revisions remain active until you disable them. If your revision doesn't scale to zero, you will pay for each active revision. As a best practice, always disable the revisions you don't need.

## Auto-scaling
Container Apps support three scale triggers: HTTP traffic, Event-driven, and Utilization (CPU or Memory usage). The utilization trigger doesn't support scale to zero. If you plan to use the KEDA event-based triggers, it is recommended to set the revision mode to single (activeRevisionMode: 'single'). The default settings (it is not clear if those are possible to change) of the auto scaler are:
* pooling interval (scale-out): 30 seconds 
* cool down interval (scale-in): 300 seconds

## Health probes
[Service supports](https://docs.microsoft.com/en-us/azure/container-apps/health-probes?tabs=arm-template) liveness, readiness and startup probes which are actually based on the standard Kubernetes health probes. You have an option to use HTTP or TCP for the probes. Make sure that your Container Apps always use health probes, which will lead to an increased overall availability of your applocations.

## Storage
**Note:** Volume mounting is in preview

The following three [storage types](https://docs.microsoft.com/en-us/azure/container-apps/storage-mounts?pivots=aca-cli) are currently supported by Container Apps: container file system, temporary storage, and Azure Files. If you need to persist data from your application, Azure Files is the only choice. At the moment of writing, the only way (at this moment) to mount storage into the running containers is by providing a YAML file with Azure CLI. This approach is not recommended for production because of the inability to provide a declarative specification for the deployment using tools like Bicep, ARM templates, or Terraform. Using a private endpoint (Private Link) to access the storage account (Azure Files) is strongly recommended. 

## Observability
Container Apps integrates out-of-the-box with the platform's native monitoring service - Azure Monitor. Everything you write to stdout and stderr from the running containers is collected and stored in the table ContainerAppConsoleLogs_CL in Log Analytics Workspace. You can use Kusto to query the records and create an [Azure Monitor log alert rules](https://docs.microsoft.com/en-us/azure/azure-monitor/alerts/alerts-log) to receive alerts when desired. For debugging purposes, you can utilize the [Log Streaming](https://docs.microsoft.com/en-us/azure/container-apps/observability?tabs=bash#log-streaming) functionality to view the stdout and stderr messages in real time and use the [Container Console](https://docs.microsoft.com/en-us/azure/container-apps/observability?tabs=bash#container-console) to connect to a container and perform a debug from inside. The retention period depends on the configured retention period of the Log Analytics Workspace - from 30 up to 730 days:

![Desktop View]({{ "/assets/img/posts/container-apps/log_retention.png" | relative_url }})

Metrics are also [stored in Azure Monitor](https://docs.microsoft.com/en-us/azure/container-apps/observability?tabs=bash#azure-monitor-metrics). In the Metrics Explorer, you can view the metrics, apply filters and splitting and create multiple charts in a single view to correlate the metrics between different Container Apps and/or services (for example, Container App + Database).

Based on the metric value, you can also [create alerts](https://docs.microsoft.com/en-us/azure/container-apps/observability?tabs=bash#azure-monitor-alerts). The default retention period is 93 days, and it is not extendable at this moment (not until the Diagnostic Settings are available for Container Apps).

# Security
## Managed Identities
Both types of managed identities are supported: System and User assigned. Because the identity can be used to get the image from the container registry, you must always use a user-assigned identity. Ensure that the user-assigned identity is pre-provisioned and has the AcrPull permission over the Azure Container Registry before deploying the Container Apps. From within your application (container), you can call the [internal REST endpoint](https://docs.microsoft.com/en-us/azure/container-apps/managed-identity?tabs=portal%2Cdotnet#connect-to-azure-services-in-app-code) to get the access token that you can use to access other OAuth2 protected services like Key Vault, Storage Account, etc. Always use the client libraries available for multiple programming languages when interacting with the REST endpoint.

## Managing Secrets
Container Apps offer [integrated secret management](https://docs.microsoft.com/en-us/azure/container-apps/manage-secrets) that you can use to store your secrets. You can use the secrets for storing connection strings for KEDA-based scale triggers or pass them as environment variables to the containers you deploy. Never store the secret value in the repository. Store the secrets as GitHub secrets, HashiCorp Vault, or Azure Key Vault. During the deployment phase,  read the value from the vault and pass it as a secure parameter value.

Because Container Apps does not support [Dapr Secrets Management API](https://docs.dapr.io/developing-applications/building-blocks/secrets/secrets-overview/) it is recommended to use the built-in secret management. Otherwise, you will need to add service-specific code to handle the secrets, which will add complexity without bringing any value.

## Built-in auth mechanism
**Note:** Auth stands for authentication + authorization

Similar to Web Apps and Function Apps, Container Apps offer a [built-in mechanism for authentication and authorization](https://docs.microsoft.com/en-us/azure/container-apps/authentication) for published applications. Rather than building your auth stack in the application, it is recommended to use the built-in one. 

When configuring an Azure Active Directory (Microsoft) as an identity provider, the Azure AD application registration can be created automatically, or you can provide an existing (pre-created) application registration. A recommendation is to pre-created application registration using the [following script](https://raw.githubusercontent.com/tosokr/container-apps-templates/main/container-apps/configure-aad-auth.sh).

The last command in the script will enable the built-in auth mechanism for the Container Apps. If you prefer to do this via the portal, all the information you need is available via variables:

![Desktop View]({{ "/assets/img/posts/container-apps/auth_portal_variables.png" | relative_url }})

After the successful addition of an identity provider, the client's (application) secret is stored in Container Apps' secret under the name microsoft-provider-authentication-secret. If you rotate the secret, make sure you update the value before you create a new revision. 

**Note:** Although the client secret is an optional parameter when using an existing application, always configure it because otherwise, you will end up using the implicit grant flow which these days is considered unsecured.

When enabled, the built-in auth deploys a sidecar container to each replica in the revision. The sidecar container handles all the aspects of authentication and authorization, meaning there is no need to change your code to support auth. If from your code you need to know who is accessing the application, you can access that information from the following headers:

* X-MS-CLIENT-PRINCIPAL-NAME
* X-MS-CLIENT-PRINCIPAL-ID

By default, any principal (user or service) can request tokens from Azure AD and access the application by presenting that token. If the behavior is not desired, you can:

* [restrict the application to a set of users](https://docs.microsoft.com/en-us/azure/active-directory/develop/howto-restrict-your-app-to-a-set-of-users)
* [configure application roles](https://docs.microsoft.com/en-us/azure/active-directory/develop/howto-add-app-roles-in-azure-ad-apps) and [implement role-based access control in your application](https://docs.microsoft.com/en-us/azure/active-directory/develop/howto-implement-rbac-for-apps)

# Reliability
## High-availability
If you deploy in a custom VNET, you have an option to enable  Availability Zones support to get a data center redundancy. In the backend, the Container Apps service creates VMSS in each Availability Zones for scheduling the applications. The Availability Zone distribution of the containers is hidden from the users. Although you have an option to disable it, as a general recommendation, always use Availability Zone deployments because they improve your overall application availability. 

From an application perspective, make sure you use at least two application replicas to provide high availability. Also, the event-based scaling for your application can improve availability. Why? Because if some of the containers are becoming unresponsive due to some reason, the auto-scaler will kick in and spin up new replicas to handle the load. 

## Geo-redundancy
An Azure region is a set of datacenters deployed within a latency-defined perimeter and connected through a dedicated regional low-latency network. Azure consists of 60+ regions around the world. Some regions are further divided into availability zones and unique physical locations, enabling redundancy at a regional level.

Under rare circumstances, it is possible that facilities in an entire region can become inaccessible, for example, due to network failures or natural disasters. To protect against such a scenario, you can apply one of the following strategies:

* **Redeploy on disaster:** In this approach, the infrastructure and application are redeployed from scratch at the time of disaster. Copy of the data is usually available in the disaster region in backups or asynchronous replications. This strategy is appropriate for non-critical applications that don’t require a guaranteed recovery time. To be successful with this strategy, your infrastructure as code scripts need to be region independent. You need to test your deployments in the disaster recovery region, for example, deploying your test environment once every month to verify that all the resource SKUs you use are also available in the DR region. 
* **Warm Spare (Active/Passive):** A secondary hosted service is created in an alternate region, and roles are deployed to guarantee minimal capacity; however, the roles don’t receive production traffic. This approach is useful for applications not designed to distribute traffic across regions.
* **Hot Spare (Active/Active):** The application is designed to receive a production load in multiple regions. The cloud services in each region might be configured for higher capacity than required for disaster recovery purposes. Alternatively, the cloud services might scale-out as necessary during a disaster and failover. This approach requires substantial investment in application design, but it has significant benefits. These include low and guaranteed recovery time, continuous testing of all recovery locations, and efficient capacity usage.

For Warm and Hot spare strategies for external environments, you can deploy Front Door or Traffic Manager to act as a global load balancer for your Container Apps applications:

![Desktop View]({{ "/assets/img/posts/container-apps/geo_redundancy.png" | relative_url }})

For internal environments, it is possible, although not recommended due to the complexity, to achieve active/passive geo-redundancy by using private DNS zones.

## Service Level Agreement (SLA)
Container Apps has a 99.95% SLA which translates to almost 22 minutes of acceptable downtime per month. The offered SLA is in line with the uptime SLA for AKS, which is also 99.95%. What is strange with the Container App's SLA is that it is independent of the usage of Availability Zones. So, with or without Availability Zones support, you will get the same SLA.

# Pricing
[Container Apps costs](https://azure.microsoft.com/en-us/pricing/details/container-apps/) are calculated based on three meters:

* vCPU seconds - number of seconds you allocate a vCPU
* Gibibyte seconds - number of seconds you allocate a memory
* Requests - number of processed requests (with enabled ingress only)

Implementing a proper scaling policy will reduce costs in all environments, especially on dev and test. If the Container Apps scale to zero, you don't pay anything because they are not allocating resources or serving requests. For Container Apps replicas in idle state (resource usage below active billing threshold), you will pay ~90% less for the vCPU allocation. You may have Container Apps (replicas) in an idle state to avoid cold starts for your application. 

Billing at seconds granularity enables cheaper instant spikes handling. Also, there is a monthly free grant (on a subscription level) which is automatically applied. 