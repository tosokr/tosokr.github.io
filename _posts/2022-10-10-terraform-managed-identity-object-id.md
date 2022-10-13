---
title: How to get the System Assigned Managed Identity Object Id from within the VM?
date: 2022-10-13 00:00:00 +0000
description: To get the Object Id of the VM's System Assigned Managed Identity, you need to call the Azure Instance Metadata Service (IMDS) endpoint and use the provided attributes to get the required information. It is more challenging when you want to achieve the same thing with Terraform.  
categories: [Security]
tags: [Managed Identity, Terraform]
toc: false
header:
 teaser: "/assets/img/posts/teasers/vm.jpg"
permalink: /security/managed-identity-vm-objectid-terraform/
excerpt: To get the Object Id of the VM's System Assigned Managed Identity, you need to call the Azure Instance Metadata Service (IMDS) endpoint and use the provided attributes to get the required information. It is more challenging when you want to achieve the same thing with Terraform.  
---

While working with Terraform and Azure DevOps self-hosted agent, I faced a challenge getting the Object(Principal) Id of a System Assigned Managed Identity to the VM running the agent. I needed that Object Id to assign access to a Key Vault. There is a way to do so via Azure CLI:

```bash
az resource list -n $(curl -s -H Metadata:true --noproxy '*' 'http://169.254.169.254/metadata/instance?api-version=2021-02-01' | jq -r .compute.name) -g $(curl -s -H Metadata:true --noproxy '*' 'http://169.254.169.254/metadata/instance?api-version=2021-02-01' | jq -r .compute.resourceGroupName) --query '{principal_id:[0].identity.principalId,tenant_id:[0].identity.tenantId}' --out json
```
but that approach requires the installation of Azure CLI and configuring Reader access on the VM itself. 

A more elegant approach is to extract the Object Id from the access token received by [calling the Azure Instance Metadata Service (IMDS) endpoint](https://learn.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/how-to-use-vm-token):

```bash
curl "http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https://management.azure.com/" --header "Metadata: true" | jq -r .
access_token | jq -R 'split(".") | .[1] | @base64d | fromjson | .oid' -r
```
In Terraform, you can use the external provider to execute the script:

```terraform
data "external" "account_info" {
  program = ["bash", "-c", "curl -s 'http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https://management.azure.com/' --header 'Metadata: true' | jq -r .access_token | jq -R 'split(\".\") | .[1] | @base64d | fromjson | {oid: .oid, appid: .appid,tid: .tid}'"]
}
```

and then access the Object Id (and the Tenant Id) with:

```terraform
tomap(data.external.account_info.result).oid
```

For example:
```terraform
resource "azurerm_key_vault_access_policy" "current-user" {
  key_vault_id = azurerm_key_vault.kv.id

  tenant_id = tomap(data.external.account_info.result).tid
  object_id = tomap(data.external.account_info.result).oid

  key_permissions = [
    "Create",
    "Delete",
    "Get",
    "Purge",
    "Recover",
    "Update",
    "List",
    "Decrypt",
    "Sign"
  ]
}
```