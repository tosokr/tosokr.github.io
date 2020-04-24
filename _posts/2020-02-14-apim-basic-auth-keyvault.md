---
title: Basic authentication in API Management using Key Vault
date: 2020-02-27 19:00:00 +0000
description: Azure API Management policy client basic authentication howto
categories: [API Management]
tags: [APIMPolicy,Powershell,Authentication]
header:
 teaser: "/assets/img/posts/teasers/apiManagement.png"
---
Policies are a powerful capability of the Azure API Management (APIM) that allows the publisher to change the behavior of the API through configuration. APIM policy is a collection of statements executed sequentially on the request or response of an API. We can define our policy statement in four configuration sections: inbound, backend, outbound, and on-error:
```xml
<policies>
  <inbound>
    <!-- statements to be applied to the request go here -->
  </inbound>
  <backend>
    <!-- statements to be applied before the request is forwarded to 
         the backend service go here -->
  </backend>
  <outbound>
    <!-- statements to be applied to the response go here -->
  </outbound>
  <on-error>
    <!-- statements to be applied if there is an error condition go here -->
  </on-error>
</policies>
```

APIM uses subscriptions for authentication and authorization to the published APIs. If we need to perform some other type of authentications, there is support for OAuth 2.0, Certificates, and Basic authentication. Although we can think of the Basic authentication as a legacy, there are still companies who are using this authentication mechanism, mostly because of its simplicity and legacy software base. 
The real challenge with the Basic authentication in APIM is the storage of the authentication details for the users (usernames and passwords). There is an option to use named values feature of the APIM to store a combination of username and password and reference those from the policy:
```xml
<choose>
	<when condition="@(context.Request.Headers.GetValueOrDefault("Authorization").AsBasic().UserId!="{{UserId}}" || context.Request.Headers.GetValueOrDefault("Authorization").AsBasic().Password!="{{Password}}")">
	    <return-response>
	        <set-status code="401" reason="Not authorized" />
	    </return-response>
	</when>
</choose>
```
The problem with this approach is that named pairs are verified during the compilation of the policy, meaning that there is no possibility to dynamically generate the name of the named pair (in short words, we can’t have put a variable in {{  }}). The solution is to store the user details in a Key Vault secret, where the name of the secret represents the username, and the secret value represents the password for that user. 

1. Set the variables for the powershel cmdlets
```powershell
$subscriptionId="" #subscription to use for the deployment
$resourceGroupName = "" #name of the resource group
$apiManagementName ="" #name of the API Management gateway
$keyVaultName ="" #name of the Key Vault
$demoUserName ="demo" #username for basic auth
$demoUserPassword= "p@ssword1" #user's password
```
2. Login to the Azure environment:
```powershell
Login-AzAccount
Set-AzContext -Subscription $subscriptionId
$resourceGroup = Get-AzResourceGroup -Name $resourceGroupName
```
3. Enable managed identity for API Management, if not enabled
```powershell
$apiManagement = Get-AzApiManagement -ResourceGroupName $resourceGroupName `
-Name $apiManagementName
if (-not $apiManagement.Identity){
    Set-AzApiManagement -InputObject $apiManagement -AssignIdentity
}
$apiManagement = Get-AzApiManagement -ResourceGroupName $resourceGroupName `
-Name $apiManagementName
```
4. Create the Key Vault, if not exists
```powershell
$keyVault = Get-AzKeyVault -VaultName $keyVaultName `
-ErrorVariable keyVaultNotPresent -ErrorAction SilentlyContinue
if (-Not $keyVault){
  $keyVault = New-AzKeyVault -Name $keyVaultName `
  -ResourceGroupName $resourceGroupName -Location $resourceGroup.Location
}
```
5. Enable access to the Key Vault secrets from the API Management
```powershell
Set-AzKeyVaultAccessPolicy -ResourceGroupName $resourceGroupName `
-VaultName $keyVault.VaultName -ObjectId $apiManagement.Identity.PrincipalId `
-PermissionsToSecrets get
```
6. Add the demo user and password into the Key Vault
```powershell
Set-AzKeyVaultSecret -Name $demoUserName -VaultName $keyVault.VaultName `
-SecretValue $(ConvertTo-SecureString -String $demoUserPassword `
-AsPlainText –Force)
```
7. Create a Named value in the API Management with the Key Vault URL
```powershell
New-AzApiManagementProperty -Context $(New-AzApiManagementContext -ResourceGroupName $resourceGroupName -ServiceName $apiManagementName) `
 -Name KeyVaultURL -Value $keyVault.VaultUri
```
8. In the inbound part of your policy definition, add the following code

```xml
<!-- If the Authorization header does not exists in the header, return 401 -->
<choose>
    <when condition="@(!context.Request.Headers.ContainsKey("Authorization"))">
        <return-response>
            <set-status code="401" reason="Unauthorized" />
        </return-response>
    </when>
</choose>
<!-- check the cache for secret first -->
<cache-lookup-value key="@(context.Request.Headers.GetValueOrDefault("Authorization").AsBasic().UserId)" variable-name="keyvaultsecretResponse" />
<!-- call Key Vault if not found in cache -->
<choose>
    <when condition="@(!context.Variables.ContainsKey("keyvaultsecretResponse"))">
        <send-request mode="new" response-variable-name="keyvaultsecret" timeout="20" ignore-error="false">
            <set-url>@{
                return "{{KeyVaultURL}}" +"secrets/" + context.Request.Headers.GetValueOrDefault("Authorization").AsBasic().UserId + "/?api-version=7.0";
            }</set-url>
            <set-method>GET</set-method>
            <authentication-managed-identity resource="https://vault.azure.net" />
        </send-request>
        <!-- transform response to string and store in cache -->
        <set-variable name="keyvaultsecretResponse" value="@((string)((IResponse)context.Variables["keyvaultsecret"]).Body.As<JObject>()["value"])" />
        <cache-store-value key="@(context.Request.Headers.GetValueOrDefault("Authorization").AsBasic().UserId)" value="@((string)context.Variables["keyvaultsecretResponse"])" duration="3600" />
    </when>
</choose>
<!--Check if password from the Authorization header matches the secret value from the Key Vault -->
<choose>
    <when condition="@(context.Request.Headers.GetValueOrDefault("Authorization").AsBasic().Password!= (string)context.Variables["keyvaultsecretResponse"])">
        <return-response>
            <set-status code="401" reason="Not authorized" />
        </return-response>
    </when>
</choose>
<!--Delete the Authorization header, because it can cause problems at the backend-->
<set-header name="Authorization" exists-action="delete" />

```
Basic authentication is a Base64 representation of the combination username:password (if you changed the username and password combination from above, use [https://www.base64encode.org](https://www.base64encode.org) to generate your Base64 string).
When calling the API, add the following header in the request:
```
Key: Authorization
Value: Basic ZGVtbzpwQHNzd29yZDE=
```

Grab a beer. You finished! 