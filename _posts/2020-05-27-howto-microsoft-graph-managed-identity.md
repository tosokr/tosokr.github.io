---
title: How to add Microsoft Graph API permissions to a Managed Identity
date: 2020-05-27 12:00:00 +0000
description: 
categories: [Security]
tags: [MicrosoftGraph, ManagedIdentity]
toc: false 
header:
 teaser: "/assets/img/posts/teasers/managedIdentities.png"
permalink: /security/microsoft-graph-permissions-managed-identity/
excerpt: Managed identities for Azure resources provide Azure services with an automatically managed identity in Azure AD. When accessing the Microsoft Graph, the managed identity needs to have proper permissions for the operation it wants to perform. At the time of writing (May 2020), there is no option to assign such permission through the Azure Portal. 
---
The biggest security challenge for every application is the storage of the credentials. The managed identities for Azure resources provide Azure services with an automatically managed identity in Azure AD. You can use the identity to authenticate to any service that supports Azure AD authentication, without any credentials in your code.
Azure Logic App has an option when connecting to an HTTP endpoint to use its managed identity for authentication:

![Desktop View]({{ "/assets/img/posts/managedIdentity/logicAppHTTPManagedIdentity.png" | relative_url }})

When accessing the Microsoft Graph, the managed identity needs to have proper permissions for the operation it wants to perform. At the time of writing (May 2020), there is no option to assign such permission through the Azure Portal: 

![Desktop View]({{ "/assets/img/posts/managedIdentity/managedIdentityPermissions.png" | relative_url }})

The Powershell script below will add the requested Microsoft Graph API permissions to the Managed Identity object: 

```powershell
# Your tenant id (in Azure Portal, under Azure Active Directory -> Overview )
$TenantID=""
# Microsoft Graph App ID (DON'T CHANGE)
$GraphAppId = "00000003-0000-0000-c000-000000000000"
# Name of the manage identity (same as the Logic App name)
$DisplayNameOfMSI="demoLogicApp" 
# Check the Microsoft Graph documentation for the permission you need for the operation
$PermissionName = "Domain.Read.All" 

# Install the module (You need admin on the machine)
Install-Module AzureAD 

Connect-AzureAD -TenantId $TenantID 
$MSI = (Get-AzureADServicePrincipal -Filter "displayName eq '$DisplayNameOfMSI'")
Start-Sleep -Seconds 10
$GraphServicePrincipal = Get-AzureADServicePrincipal -Filter "appId eq '$GraphAppId'"
$AppRole = $GraphServicePrincipal.AppRoles | `
Where-Object {$_.Value -eq $PermissionName -and $_.AllowedMemberTypes -contains "Application"}
New-AzureAdServiceAppRoleAssignment -ObjectId $MSI.ObjectId -PrincipalId $MSI.ObjectId `
-ResourceId $GraphServicePrincipal.ObjectId -Id $AppRole.Id
```

After executing the script, in the portal, the requested API permissions are assigned to the Managed Identity: 
![Desktop View]({{ "/assets/img/posts/managedIdentity/managedIdentityPermissionsAdded.png" | relative_url }})
