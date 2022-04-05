---
title: Deep dive into Defender for Containers
date: 2022-04-05 00:00:00 +0000
description: Microsoft Defender for Containers is the new plan that merges the capabilities of the two existing Microsoft Defender for Cloud plans, Microsoft Defender for Kubernetes and Microsoft Defender for container registries, and adds a new set of features like multi-cloud support, Kubernetes-native deployment, Advanced Threat Detection and vulnerability assessment
categories: [Azure Kubernetes Service, Security]
tags: [Kubernetes, Defender]
toc: true
header:
 teaser: "/assets/img/posts/teasers/defender.png"
permalink: /security/defender-for-containers/
excerpt: Microsoft Defender for Containers is the new plan that merges the capabilities of the two existing Microsoft Defender for Cloud plans, Microsoft Defender for Kubernetes and Microsoft Defender for container registries, and adds a new set of features like multi-cloud support, Kubernetes-native deployment, Advanced Threat Detection and vulnerability assessment
---

# Overview

Microsoft Defender for Containers is the new plan that merges the capabilities of the two existing Microsoft Defender for Cloud plans, Microsoft Defender for Kubernetes and Microsoft Defender for container registries, and adds a new set of features:

* Multi-cloud support: AKS  and any Cloud Native Computing Foundation (CNCF) certified Kubernetes clusters (through Azure Arc)
* Kubernetes-native deployment: automatic deployment using DaemonSet
* Advanced Threat Detection: deterministic, AI, and anomaly-based detection
* Vulnerability assessment: continuous scan for running images

# Provisioning

First, the plan (the product)  needs to be provisioned on the subscription via the portal or using the REST API:

```bash
az rest --method PUT \
    --uri "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/providers/Microsoft.Security/pricings/Containers?api-version=2018-06-01" \
    --body '{"properties": {"pricingTier": "standard"}}'
```

or using CLI:

```bash
az security pricing create --name Containers --tier Standard --subscription $SUBSCRIPTION_ID
```

After enabling the plan, the components can be provisioned automatically or manually on the clusters. The provisioning is related to two of the components in Defender for Containers:

* Defender security profile (the DaemonSet): Security profile deployed to each worker node, collects security-related data and sends it to Defender for analysis. It is required for runtime protections and security capabilities provided by Defender for Containers.
* Azure policy for Kubernetes add-on: Extends Gatekeeper v3, required to apply at-scale auditing, enforcements, and safeguards on clusters in a centralized, consistent manner. The policy add-on is not an exclusive feature of Defender for Containers, but it is an integral part of the overall AKS security solution.

## Automatic

The automatic provisioning is enabled using Azure Policies:

* [Configure Azure Kubernetes Service clusters to enable Defender profile](https://portal.azure.com/#blade/Microsoft_Azure_Policy/PolicyDetailBlade/definitionId/%2Fproviders%2FMicrosoft.Authorization%2FpolicyDefinitions%2F64def556-fbad-4622-930e-72d1d5589bf5) - uses a System assigned managed identity with Contributor and Log Analytics Contributor roles to remediate the cluster.
* [Deploy Azure Policy Add-on to Azure Kubernetes Service clusters](https://portal.azure.com/#blade/Microsoft_Azure_Policy/PolicyDetailBlade/definitionId/%2Fproviders%2FMicrosoft.Authorization%2FpolicyDefinitions%2Fa8eff44f-8c92-45c3-a3fb-9880802d67a7) - uses a System assigned managed identity with Azure Kubernetes Service Contributor Role role to remediate the cluster.

Because it is based on Azure Policies, the automatic provisioning will work only for clusters that have enabled Azure RBAC (something not mentioned in the documentation). The two policies can be applied on the Management Group level, coupled with a custom policy that enables Azure RBAC on deployed clusters for an automated Defender for Container provisioning across the tenant. 

It can happen that both policies will try to update the cluster simultaneously. In that situation, one of the two components will not be deployed into the cluster with the following error displayed under _Deployments_ in the cluster's resource group: _Operation is not allowed: Another operation (systemtemp - Updating) is in progress, please wait for it to finish before starting a new operation. See https://aka.ms/aks-pending-operation for more details_. 

A policy remediation task is needed to protect against end-users deleting the components deployments and fixing the above issue.


## Manual

The manual deployment can be done through the portal (by fixing a recommendation in Defender) or using the REST API:

```bash
az rest --method PUT \
    --uri "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourcegroups/$RESOURCE_GROUP/providers/Microsoft.ContainerService/managedClusters/$CLUSTER_NAME?api-version=2022-01-01" \
    --body '{"location": "'$LOCATION'","properties": {"securityProfile": {"azureDefender": {"enabled": true,"logAnalyticsWorkspaceResourceId": "'$logAnalyticsWorkspaceResourceId'"}}}}'
```

# Features
## Scanning images for vulnerability

Defender for Containers scans images for vulnerabilities stored in an ACR. It pulls the image from the registry and runs it in an isolated sandbox with the Qualys scanner. Image scan is triggered for every image that is pushed, pulled, or imported into the registry. In addition, the recently (in the last 30 days) pulled images are scanned weekly. If the defender security profile (the DaemonSet) is deployed in a cluster, it will initiate a weekly scan of the running images pulled from ACR.

The vulnerabilities findings are [listed under Recommendations](https://docs.microsoft.com/en-us/azure/defender-for-cloud/defender-for-container-registries-usage#view-and-remediate-findings) and are visible a few seconds after the scan finishes. 

Available in private preview, there is a policy named [Kubernetes clusters should gate deployment of vulnerable images](https://portal.azure.com/#blade/Microsoft_Azure_Policy/PolicyDetailBlade/definitionId/%2Fproviders%2FMicrosoft.Authorization%2FpolicyDefinitions%2F13cd7ae3-5bc0-4ac4-a62d-4f7c120b9759) which can deny the deployment of images with vulnerabilities identified by the CI/CD scanning or ACR scanning. To make use of it, the Azure Policy add-on must be installed on the cluster.

There is a possibility to [disable specific findings](https://docs.microsoft.com/en-us/azure/defender-for-cloud/defender-for-container-registries-usage#disable-specific-findings) based on multiple parameters, including severity and CVSS (Common Vulnerability Scoring System) score. That will prevent unnecessary noise and enable us to focus only on the findings that need action. 

## CI/CD integration

For Github workflows, there is a Azure/container-scan action that scans the images for vulnerabilities using Trivy and performs static code analysis using Dockle. The results from the scans can be pushed to the Defender for Containers using an Application Insight. The complete guideline is available [here](https://docs.microsoft.com/en-us/azure/defender-for-cloud/defender-for-container-registries-cicd). 

As it is documented, the integration can improve visibility. It only makes sense if it is combined with the policy [Kubernetes clusters should gate deployment of vulnerable images](https://portal.azure.com/#blade/Microsoft_Azure_Policy/PolicyDetailBlade/definitionId/%2Fproviders%2FMicrosoft.Authorization%2FpolicyDefinitions%2F13cd7ae3-5bc0-4ac4-a62d-4f7c120b9759). Another option is to parse the output and not push to the ACR images with specific vulnerabilities or best practices violations.

## Runtime security

Defender for Containers provides real-time threat protection and generates alerts for suspicious activities. Threat protection at the cluster level is provided by the Defender profile (the DaemonSet) and analysis of the Kubernetes audit logs.

The node-level threat detection, DaemonSet that runs on all of the nodes in the cluster, inspects activity on the Kubernetes worker-node to detect suspicious activity that runs by the containers on the nodes. The solution is developed for Kubernetes and provides container-specific alerts and vulnerabilities detections as they are discovered. It also tracks the [MITRE ATT&CK®](https://attack.mitre.org/matrices/enterprise/containers/) matrix for Containers, which is a collection of tactics and techniques of attacks related to container environments. 

The Defender profile deploys the following components into the cluster:

![Desktop View]({{ "/assets/img/posts/defender/containers_table.png" | relative_url }})

Although initial resource requests are low, proper testing is needed because the total max limits for the DaemonSets per node are: 270m CPU and 328Mi memory.

When Defender for Container is enabled on a cluster, it will monitor the Kubernetes API operations to find suspicious and malicious activities in the Kubernetes control plane. Under the hood, Defender collects the API logs over the Azure backbone, analyzes those, and presents the findings. The process of collecting and analyzing is transparent and there is no cost associated with storing the API logs.

## Security Alerts

Microsoft Defender for Containers provides [security alerts](https://docs.microsoft.com/en-us/azure/defender-for-cloud/alerts-reference#alerts-k8scluster) categorized in three severity levels: High, Medium, and Low. Fired alerts are visible in the Security alerts sections in Defender for Cloud.  [Email notifications](https://portal.azure.com/#blade/Microsoft_Azure_Security/SecurityMenuBlade/EnvironmentSettings) are also configurable, based on the severity level, to the chosen email recipients (by default to the owners of the subscription). Microsoft has an [add-on](https://splunkbase.splunk.com/app/4564/) that uses [Microsoft Graph Security API](https://docs.microsoft.com/en-us/graph/api/resources/security-api-overview?view=graph-rest-1.0) to ingest the security alerts into Splunk. The security alerts and recommendations are also exportable to an Event Hub from where they can be consumed by Splunk using the  [Splunk add-on for Microsoft Cloud Services](https://splunkbase.splunk.com/app/3110/#/overview).  

# Pricing

Microsoft Defender for Container is list priced $7 per vCore associated with a Kubernetes cluster. Please note that vCore differs from vCPU for the SKUs that support hyper-threading (D series, for example), and the ratio is 2:1 (vCPU:vCore). The average number of vCores in the last 30 days is taken into account for the monthly calculations. The price also includes 20 free ACR vulnerabilities scans per vCore. Any additional scan is list priced at $0.29 per image digest.

A workbook for cost estimation across multiple subscriptions is available on [GitHub](https://github.com/Azure/Microsoft-Defender-for-Cloud/tree/main/Workbooks/Defender%20for%20Containers%20Cost%20Estimation). The workbook supports a maximum of 200 subscriptions. To generate the cost estimate for bigger tenants, use the following Resource Graph Query:

```
resources
| where type =~ "microsoft.containerservice/managedclusters"
| extend machineType = tostring(properties.agentPoolProfiles[0].vmSize), nodesCount = toint(properties.agentPoolProfiles[0]['count']), powerState = tostring(properties.agentPoolProfiles[0].powerState.code), minMachineCount = toint(properties.agentPoolProfiles[0].minCount), maxMachineCount = toint(properties.agentPoolProfiles[0].maxCount)
| extend vCoresPerMachine = toint(extract_all(@"(\d+)", machineType)[0])
| extend coresTotal = vCoresPerMachine * nodesCount
| extend minMonthlyCost = vCoresPerMachine * minMachineCount * 7, maxMonthlyCost = vCoresPerMachine * maxMachineCount * 7, estimatedMonthlyCost = coresTotal * 7
| summarize sum(estimatedMonthlyCost)
```

# Conclusion

Defender for Containers can significantly improve the security of the container environments. The solution can be automatically provisioned to the newly created and already existing clusters through the (Azure) platform. People with Security admin and Security reader roles on the subscriptions can view their security recommendations and alerts and have an active role in the security observation of the deployed resources. A single support contact point covers both the product and the underlying infrastructure in case of an issue. 

Pushed, pulled, and imported images to ACR will be scanned for vulnerabilities with Qualys. On top of it, if used, the CI/CD integration will add scan results based on Trivy and security best practices.  With the runtime integration, the images running in Kubernetes will be scanned every week for any newly discovered vulnerabilities. At this moment, there is no support for scanning running images pulled from public repositories. Microsoft has that feature in the backlog.

The runtime security doesn't provide an inventory list of pods running in the cluster. For environments with many clusters, having that central list of running images accross the tenant is very useful. Today, to get that inventory list yet another agent need to be deployed into the cluster - the Log Analytics agent.

The Azure Policy-addon for Kubernetes is required in order to get the maximum out of Defender for Containers, like denying deployment of vurnelable images. The policy add-on will apply all of the policies applicable to the resource, coming from the management group, subscription and resource group. This can lead to an unexpected behaivour, because Kubernetes related policies are part of different policy initiatives that maybe are already applied at the subscription level.
