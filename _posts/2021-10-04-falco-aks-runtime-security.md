---
title: Falco as an AKS runtime security tool
date: 2021-10-04 00:00:00 +0000
description: Falco is an open-source tool for container runtime security that can help you secure AKS from zero-day vulnerabilities and unexpected behaviors inside containers and in the host OS. Using Flacosidekick, you can add custom fields to the generated events and forward those to your ecosystem of observability and SIEM tools.
categories: [Azure Kubernetes Service]
tags: [AKS]
toc: true 
header:
 teaser: "/assets/img/posts/teasers/falco.png"
permalink: /aks/falco-aks-runtime-security/
excerpt: Falco is an open-source tool for container runtime security that can help you secure AKS from zero-day vulnerabilities and unexpected behaviors inside containers and in the host OS. Using Flacosidekick, you can add custom fields to the generated events and forward those to your ecosystem of observability and SIEM tools. 
---

# Overview

[Falco](https://falco.org), the cloud-native runtime security project, is the de facto Kubernetes threat detection engine that enables to tackle:

* Zero-day vulnerabilities – issues previously unknown to a software creator
* Privilege escalation attempts
* Bugs that cause erratic behavior or resource leaking
* Unexpected behavior in the deployed artifacts

It is the first runtime security project to join the CNCF as an incubation-level project.  Falco detects unexpected or malicious application and environment behavior using security rules. Rules are built-in, available from the community, and easily extendable using Falco’s flexible rules engine. Falco is easily pluggable into the security detection and response workflows to identify and alert on runtime threats.

Falco, by default, emits events in text or JSON format to stdout and syslog and has an option to save those events (security notification) in a file.  The behavior is changeable via the configuration. The emitted events can be collected by the SIEM tool for real-time analysis and reporting. 

# Falco in AKS

Falco is a DaemonSet that deploys a pod to each node in the cluster. Using the libscap and linsinsp libraries it parses events from the underlying OS and translates those into human-readable strings. 

![Desktop View]({{ "/assets/img/posts/falco/falco.png" | relative_url }})

By default, the Falco pod has configured resource limits to make sure that it will not consume all of the node resources in case of misconfiguration:

```yaml
Limits:
    cpu:     1
    memory:  1Gi
Requests:
    cpu:        100m
    memory:     512Mi
```

When deployed in AKS, Falco can't access and process the Kubernetes Audit logs because those, from the nodes Microsoft manages, can be exported only to: Storage Account, Event Hub, Log Analytics Workspace or any of the partner solutions: Datadog, Elastic, Logz.io and Apache Kafka for Confluence Cloud. In theory, you can create a k8s deployment that downloads those events and forward them to Falco via the webserver. A more reasonable solution will be to connect the AKS audit log destination directly to the SIEM tool and process the events there.

# Connect Falco to your ecosystem with Falcosidekick

[Falcosidekick](https://github.com/falcosecurity/falcosidekick) is a simple tool used to connects Falco to the ecosystem. It takes Falco's events and forwards them to different output in a fan-out way. The one significant advantage of using Falcosidekick is the custom fields that you can add to the events. Kubernetes cluster name is not included in the events fields, so to identify from which cluster the event is coming, you need to add a custom field using Falcosidekick. 
When during the Helm deployment, Falcosidekick support is enabled for Falco, the deployment will automatically configure _json_output: true_ and _json_include_output_property: true_ for Falco, and also will enable http output for the events to the Falcosidekick service:

```yaml
http_output:
  enabled: true
  url: http://falco-falcosidekick:2801
```

Falcosidekick has built-in support for Azure Event Hub, which can be further consumed by different SIEM tools, such as Azure Sentinel or Splunk. Falcosidekick uses Managed Identity for authenticating to the Event Hub by leveraging aad-pod-identity. The solution using Event Hub will look like this:

![Desktop View]({{ "/assets/img/posts/falco/falcosidekick.png" | relative_url }})

If you are using Prometheus and Grafana as an obeservability stack for your cluster, you can: 

* use [falco-exporter](https://github.com/falcosecurity/falco-exporter) to export events directly from Falco to Prometheus (you need to enable gRPC output for Falco)
* scarp the /metrics endpoint from Falcosidekick for Falco events and metrics about Falcosideckick
* use [Grafana dashboard](https://grafana.com/grafana/dashboards/11914) to visualize Falco events

[Falcosidekick-UI](https://github.com/falcosecurity/falcosidekick-ui) is a simple WebUI for displaying the latest events from Falco locally. To view the the events, open a connection from your host to the falcosidekick-ui pod, using the _kubectl port-forward_ command. For example:

```bash
 kubectl port-forward -n falco falco-falcosidekick-ui-8hsh856769-99h8s 2802:2802
```

# Sample deplyoment script for Falco in AKS

Falco offers a [Helm chart](https://github.com/falcosecurity/charts/tree/master/falco) for easy deployment. It includes a [default ruleset](https://github.com/falcosecurity/charts/tree/master/falco/rules), which is easily extendable with custom rules. 

The following is an example script of how you can set up Falco in your AKS cluster, add custom rules, and publish the events into an Event Hub for further ingestion and processing by a SIEM tool:

```bash
# SET THE VARIABLES FOR THE DEPLOYMENT
export AKS_RESOURCE_GROUP="rg-reference2-westeurope-dev"
export AKS_CLUSTER_NAME="aks-reference2-westeurope-dev"
export DEPLOYER="ccoe" # contact details of the team operating the cluster
export IDENTITY_NAME="id-ccoe-aleksandar-falco-eventhub"
export NAMESPACE="falco"
export RESOURCE_GROUP="rg-event-hub"
export LOCATION="westeurope"
export EVENT_HUB_NAMESPACE="ttfalcotest" # needs to be globaly unique
export EVENT_HUB_NAME="falco"
export SUBSCRIPTION_ID=$(az account show --query id -o tsv)
export TENANT_ID=$(az account show --query tenantId -o tsv)

# USE A MANAGED IDENTITY IN THE NODE RESOURCE GROUP
NODE_RESOURCE_GROUP=$(az aks show -g $AKS_RESOURCE_GROUP -n $AKS_CLUSTER_NAME --query nodeResourceGroup -o tsv --only-show-errors)

# IF OF MANAGED IDENTITY ASSIGNED ON THE NODE POOLS
CLUSTER_IDENTITY_ID=$(az aks show -g ${AKS_RESOURCE_GROUP} -n ${AKS_CLUSTER_NAME} --query identityProfile.kubeletidentity.clientId -o tsv)

# CREATE USER MANAGED IDENTITY FOR THE POD TO ACCESS EVENT HUB
resourceGroupExists=$(az group exists --name "${RESOURCE_GROUP}")
if [ "${resourceGroupExists}" == "false" ]; then 
  az group create --name ${RESOURCE_GROUP} --location ${LOCATION}
fi
az identity create -g ${RESOURCE_GROUP} -n ${IDENTITY_NAME}
IDENTITY_CLIENT_ID="$(az identity show -g ${RESOURCE_GROUP} -n ${IDENTITY_NAME} --query clientId -otsv)"
IDENTITY_RESOURCE_ID="$(az identity show -g ${RESOURCE_GROUP} -n ${IDENTITY_NAME} --query id -otsv)"

# CREATE THE EVENT HUB AND ASSIGN THE ROLE FOR THE MANAGED IDENTITY
az eventhubs namespace create --name ${EVENT_HUB_NAMESPACE} --resource-group ${RESOURCE_GROUP} -l ${LOCATION}
az eventhubs eventhub create --name ${EVENT_HUB_NAME} --resource-group ${RESOURCE_GROUP} --namespace-name ${EVENT_HUB_NAMESPACE}
az role assignment create --role "Azure Event Hubs Data Sender" --assignee $IDENTITY_CLIENT_ID \
--scope /subscriptions/${SUBSCRIPTION_ID}/resourcegroups/${RESOURCE_GROUP}/providers/Microsoft.EventHub/namespaces/${EVENT_HUB_NAMESPACE}/eventhubs/${EVENT_HUB_NAME}

# INSTALL AAD-POD-IDENTITY HELM CHART AND CONFIGURE THE RBAC PREREQUISITES
helm repo add aad-pod-identity https://raw.githubusercontent.com/Azure/aad-pod-identity/master/charts
helm install --name aad-pod-identity aad-pod-identity/aad-pod-identity
az role assignment create --role "Virtual Machine Contributor" --assignee ${CLUSTER_IDENTITY_ID} \
--scope /subscriptions/${SUBSCRIPTION_ID}/resourcegroups/${NODE_RESOURCE_GROUP}
az role assignment create --role "Managed Identity Operator" --assignee ${CLUSTER_IDENTITY_ID} \
--scope /subscriptions/${SUBSCRIPTION_ID}/resourcegroups/${RESOURCE_GROUP}/providers/Microsoft.ManagedIdentity/userAssignedIdentities/${IDENTITY_NAME}

# INSTALL FALCO HELM CHART
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update

# GENERATE THE CUSTOM RULES FILE
./rules2helm.sh custom-rules/ingress-traefik.yaml custom-rules/ingress-nginx.yaml \
custom-rules/admin-activities.yaml custom-rules/file-integrity.yaml \
custom-rules/ssh-connections.yaml > custom-rules.yaml

helm upgrade --create-namespace --install -n falco falco falcosecurity/falco  \
--set webserver.enabled=false \
--set falcosidekick.enabled=true --set falcosidekick.webui.enabled=true \
--set falcosidekick.config.customfields="team:${DEPLOYER}\,clusterName:${AKS_CLUSTER_NAME}" \
--set falcosidekick.config.azure.subscriptionID=${SUBSCRIPTION_ID} --set falcosidekick.config.azure.resourceGroupName=${RESOURCE_GROUP} \
--set falcosidekick.config.azure.podIdentityClientID=${IDENTITY_CLIENT_ID} --set falcosidekick.config.azure.podIdentityName=${IDENTITY_NAME} \
--set falcosidekick.config.azure.eventHub.namespace=${EVENT_HUB_NAMESPACE} --set falcosidekick.config.azure.eventHub.name=${EVENT_HUB_NAME} \
--values custom-rules.yaml
```

# Check the generated events

To check that Falco can generates events using the custom rules, we will:
* deploy an Nginx pod 
* connect to the pod using a terminal
* start package manager inside the pod


