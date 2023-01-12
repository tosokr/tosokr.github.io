---
title: Managed Identities vs Service Principals - when to use what ?
date: 2023-01-12 00:00:00 +0000
description: Managed Identities eliminate the need for users to manage credentials by providing an identity for the Azure resource in Azure AD and using it to obtain Azure Active Directory (Azure AD) tokens. In scenarios when Managed Identies are not supported, you must use service principals. When to use what?
categories: [Security]
tags: [Managed Identity]
toc: true
header:
 teaser: "/assets/img/posts/teasers/managedIdentities.png"
permalink: /security/managed-identities-vs-service-pricipals/
excerpt: Managed Identities eliminate the need for users to manage credentials by providing an identity for the Azure resource in Azure AD and using it to obtain Azure Active Directory (Azure AD) tokens. In scenarios when Managed Identies are not supported, you must use service principals. When to use what?  
---

On this page, you can find an overview of the identity types available in Azure and a recommendation on when to use what. At the end, there is a section with an example implementation of the best practices. 

**Key takeaway**: Always preffer Managed Identities over service principals. 

# Identity types
## Managed Identities

Managed Identities eliminate the need for users to manage credentials by providing an identity for the Azure resource in Azure AD and using it to obtain Azure Active Directory (Azure AD) tokens.

Top 3 benefits of using Managed Identities:

* Eliminate the need for users to manage credentials
* Nonexportable certificate-based credentials, valid for 90 days, rolled after 45 days
* Uses long-lived tokens (24 hours) with a proactive token refresh every 12 hours (it can handle a maximum Azure AD downtime between 12-24 hours)

A Managed Identity is an Azure Resource Manager (ARM) object you create in a resource group. During the creation, you specify the location for the Managed Identity, which is just for storing the metadata. Managed Identities are special service principals created in Azure Active Directory (AAD). The Managed Identities will continue to work if there is a region outage. You can use Managed Identities only with Azure Resources to access other Azure Resources (VM access to KeyVault, for example), or any other OAuth2 protected endpoint.

You can create two types of Managed Identities:

### System-assigned Managed Identities
The Managed Identity is tied to the lifecycle of the resources, which means that if you delete the resource, the Managed Identity will also be deleted. For that reason, it is not visible in the portal as a separate resource.

Use System-assigned Managed Identities when:

* you want to be 100% sure that only that and no other resource can access a particular target
* you don't need to pre-provision the access before you create the Azure resource with enabled System-assigned Managed Identity

### User-assigned Managed Identities
The Managed Identity is created as a separate resource in Azure.

Use User-assigned Managed Identities when:

* you want to assign the same identity to multiple Azure resources
* you need to pre-provision the access before you create the Azure resource
* the principal you use for the deployment of your application doesn't have an owner or user access administrator rights (which is a best practice)

As a best practice, use a separate deployment stack to create User-assigned Managed Identities in a dedicated resource group. Also, ensure you fine-grain the managed identity permissions and don't use a single Manage Identity for everything you deploy. A good practice is to have a single identity per workload. The minimal RBAC role to be able to assign them to an Azure resource is [Managed Identity Operator](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles#managed-identity-operator). 

## Service principals
Although Managed Identity is a special type of service principal, we usually refer to a service principal as a local representation, or application instance, of a global application object in a single tenant or directory. A service principal object is created when an application is given permission to access resources in a tenant (upon registration or consent). A service principal is created automatically when you register an application using the Azure portal. You can also create service principal objects in a tenant using Azure PowerShell, Azure CLI, Microsoft Graph, and other tools. A good overview of the differences between application ↔ service principal is given on the [following page](https://learn.microsoft.com/en-us/azure/active-directory/develop/app-objects-and-service-principals). 

In some scenarios you must use service principals because Managed Identities are not supported. Keep in mind that you need to regulary rotate the secrets. One good practice is to create a new secret with each deployment as a step in your release pipeline. Check the example GitHub action bellow that can help you achiving this.  

# When to use what?
## GitHub Actions
At the end of 2021, GitHub added OpenID Connect (OIDC) support to GitHub Actions. OIDC allows you to use [Federated Credentials](https://docs.microsoft.com/en-us/graph/api/resources/federatedidentitycredentials-overview?view=graph-rest-beta) for your service principal, **without requiring a secret** and its subsequent rotation. Detailed steps to set this up are in the [GitHub](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-azure) and [Azure](https://docs.microsoft.com/en-us/azure/developer/github/connect-from-azure?tabs=azure-portal%2Clinux#use-the-azure-login-action-with-openid-connect) documentation. With the federated identity in place, you no longer need to worry about secret rotations.  

## Azure DevOps
From Azure DevOps, you can connect to Azure using Azure Resource Manager service connections. There are several types of connections:

* Service principal (automatic): Azure DevOps automatically create a service principal with a valid secret for **two years**. To rotate the secret, you must edit the service connection and click Verify.
* Service principal (manual): you manually create the service principal and assign it to the service connection. To rotate the secret, you need to generate a new secret in Azure AD and update the service connection.
* Managed identity: you need to run your own Azure DevOps agents in Azure VMs with assigned managed identities. For this service connection, you don't need to rotate the secrets.
* Publish profile: it uses a publish profile to deploy to a WebApp
* "Implicit": this creates a service principal behind the scenes when choosing service connection types that connect to an Azure subscription. Examples are:
    * Docker Registry (when choosing the Azure Container Registry subtype)
    * Kubernetes (when choosing the Azure Kubernetes Service subtype)

As a general best practice, deploy a self-hosting agents in Azure and use Managed Identities.

## Virtual Machines/Virtual Machine Scale Sets
If you need to access another Azure service (including databases) from within the Azure VM, you can use Managed Identities almost in all scenarios. The same applies if you need to access any OAuth2 endpoint (for example, an application or service). 

Under the hood, you will use the Azure Instance Metadata Service to get the OAuth2 token for the assigned Managed Identity and present that token to the endpoint you call. Check the [Microsoft documentation](https://learn.microsoft.com/en-gb/azure/active-directory/managed-identities-azure-resources/how-to-use-vm-token) on how that works and how you can grab the token using different programming languages. 

## Azure PaaS services
Almost all of the Azure PaaS Services support Managed Identities. As a general best practice, always use them when accessing any other OAuth2 endpoint, including other Azure services. 

For example, in Logic Apps, use the Managed Identity to access a storage account or a Log Analytics workspace. If that is not possible with the built-in connector, check the Rest APIs because they usually support the OAuth2 authorization workflow. 

## Promitor
[Promitor](https://docs.promitor.io/v2.8/) is an Azure Monitor scraper that makes the metrics available for Prometheus. When configuring, make sure you [use a Managed Identity to connect to Azure Monitor](https://docs.promitor.io/v2.8/scraping/runtime-configuration/#authentication).

## Grafana
Grafana also supports a Managed Identity when connecting to an Azure Monitor data source. To make use, you need to [enable the feature in the configuration](https://grafana.com/docs/grafana/v8.5/datasources/azuremonitor/#configuring-using-managed-identity).

If you use Azure Active Directory as an identity provider for Grafana, you must use a service principal and add its secret to the Grafana configuration file. This means that you must rotate that secret. As mentioned above, you create a new secret with each deployment as a step in your release pipeline and redeploy Grafana every 3 months (for example). Check the example GitHub action bellow that can help you achiving this.  

## Azure Kubernetes Service (AKS)
### Cluster & Kubelet Identity
AKS needs its identity to create additional resources like load balancers and managed disks or fetch container images from an Azure Container Registry. Always use a Managed Identity for both cluster and kubelet identities. If you have a running cluster that uses service principals, you can convert it to Managed Identity by following [this procedure](https://learn.microsoft.com/en-us/azure/aks/use-managed-identity#update-an-aks-cluster-to-use-a-managed-identity).

If you often redeploy the clusters (blue/green deployments) to an existing VNET, using User Assigned Managed Identities for both cluster and kublet makes more sense because you can use the same identity for all your deployments without the need to pre-provision the permissions with each deployment:

```bash
az aks create -g MyResourceGroup -n MyManagedCluster --assign-identity <control-plane-identity-resource-id> --assign-kubelet-identity <kubelet-identity-resource-id>
```

### Pod identities
As a best practice, you should not use fixed credentials within pods or container images, as they are at risk of exposure or abuse. Instead, assign a managed identity to pods to automatically request access to resources using a central Azure AD identity solution. [AAD Pod Identity](https://aztoso.com/aks/baseline-part-4/) enables Kubernetes applications to access cloud resources securely with Azure Active Directory. Using Kubernetes primitives, administrators configure identities and bindings to match pods. Then without any code modifications, your containerized applications can leverage any resource in the cloud that supports AAD as an identity provider. 

Always use user-managed identities (type 0) when you create AzureIdentity definitions. Make sure those identities follow the lifecycle of the applications or the AKS cluster. Using service principals is strongly not recommended.

### Azure AD Workflow identities
AAD workflow identity integrates with the Kubernetes native capabilities to federate with external identity providers. It is the next big thing for handling identities in AKS. In a nutshell, it enables you to exchange the Kubernetes service account token for an OAuth2 token representing a Managed Identity. More details about how it works are available in the [documentation](https://learn.microsoft.com/en-us/azure/aks/workload-identity-overview). 

## Kafka (Event Hubs)
Use a Managed Identity to connect to an OAuth2 protected listener in Kafka. (This repo)[https://github.com/Azure/azure-event-hubs-for-kafka/tree/master/tutorials/oauth/java/managedidentity] provides an example of how you can build consumer and producer applications in java that connects to an Azure Event Hub using the Kafka protocol. 

# Best practices in action
## Access HashiCorp Vault using Managed Identities
Enable Azure Auth on your HashiCorp namespace and allow access from the managed identity by assigning the policy. It is highly recommended  to use User Assigned Managed Identity because they can reuse and assign the same identity to multiple resources:

```bash
# Login TO THE VAULT
vault login -method=oidc # make sure you login as a namespace admin
 
# Enable Azure Auth
vault write auth/azure/config \
    tenant_id=<YOUR_AAD_TENANT_ID> \
    resource=https://management.azure.com
 
# Enable a KV-V2 secret engine to store an example secret at a custom path
vault secrets enable -path=example-secrets kv-v2
 
# Add an example secre into the vault
vault kv put example-secrets/example-password password=password1234
 
# Create an example policy
vault policy write example-policy \
    path "example-secrets/data/example-password" {
    capabilities = ["read"]
}
 
# Create a role for the Managed Identity. IDENTITY_PRINCIPAL_ID = the object (principal) id of the Managed Identity
 vault write auth/azure/role/example-role \
    policies="example-policy" \
    bound_service_principal_ids="${IDENTITY_PRINCIPAL_ID}"
```
## Get secrets from Azure Key Vault with Ansible using Managed Identities
Using Managed Identity, you can use the [azure_keyvault_secret lookup](https://github.com/ansible-collections/azure/blob/dev/plugins/lookup/azure_keyvault_secret.py) to get a secret from an Azure Key Vault. A sample implementation is provided by the CIT-DSP team here.

## GitHub Actions sample use cases
### Upload files to a storage account with federated identity (no secret/storage access key)
Ensure that the service principal you use in the azure/login action has at least the [Storage Blob Data Contributor](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles#storage-blob-data-contributor) Azure RBAC role on the storage container.

```bash
name: Upload file to a storage account
on: [push]
 
permissions:
      id-token: write
      contents: read
       
jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
    - name: 'check the code'
      uses: actions/checkout@v2
    - name: 'Az CLI login'
      uses: azure/login@v1
      with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}         
          allow-no-subscriptions: true
   
    - name: 'Run Azure CLI commands'
      run: |        
         az storage blob upload-batch --auth-mode login -d 'https://tttosokrtest.blob.core.windows.net/upload' -s .
```

### Rotate service principal's secrets with federated identity
Check the [GitHub action](https://github.com/marketplace/actions/rotate-azure-active-directory-service-principal-secret) I build for this purpose.