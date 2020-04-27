---
title: Azure Resource Manager templates - a lesson learned  
date: 2020-04-13 11:00:00 +0000
description: How to add Web App outbound IP addresses into SQL Server firewall using Azure Resource Manager template 
categories: [ARM Templates]
tags: [ARM]
header:
 teaser: "/assets/img/posts/teasers/resourceGroup.png"
permalink: /arm templates/arm-nightmare-reference/
---
Last week I worked on an ARM template for a deployment that, among other resources, included Web Apps and SQL databases. One of the tasks was to allow the connection from the possible outbound IP addresses of the Web App into the SQL server. And there, my problem started, because I'm not very proficient in creating advanced ARM templates. In the paragraphs below, I will give a brief overview of the ARM templates, what was my problem, and how did I solve it.

#### Azure Resource Manager templates
>"The Azure Resource Manager (ARM) template is a JavaScript Object Notation (JSON) file that defines the infrastructure and configuration for your project. The template uses declarative syntax, which lets you state what you intend to deploy without having to write the sequence of programming commands to create it. In the template, you specify the resources to deploy and the properties for those resources." – [docs.microsoft.com]( https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/overview)

The most significant advantage of ARM templates over scripts is that they are by design idempotent, meaning you will get the same result whenever you deploy the template, something that you want when you use release pipelines to deploy your solution. You can also create the deployment scripts (CLI and Powershell) in such a way that they are idempotent, but that usually involves a considerable overhead. 

The ARM template consists from the following sections:
-	Parameters – generalize the template by providing values during the deployment.
-	Variables -  reusable values in the template.
-	User-defined functions - customized functions to simplify the template.
-	Resources - resources to deploy.
-	Outputs - return values from the deployed resources.

#### The deployment
In the json snippets below, I'm showing the relevant parts from the deployment template. You can find the full templates in my Github repository [here.](https://github.com/tosokr/Azure/tree/master/arm/reference)

The ARM template takes two parameters: authorName and location. For simplicity, I added default values for the two.
```json
"parameters": {
    "authorName": {
        "type": "string",
        "defaultValue": "tosokr",
        "metadata": {
            "description": "Name to use for generating the resource names relevant for this template"
        }
    },
    "location": {
        "type": "string",
        "defaultValue": "[resourceGroup().location]",
        "metadata": {
            "description": "Location where to create the resources"
        }
    }
}
```
In the variables section, I'm generating a unique names for my resources, because Web App and SQL server need to have globally unique names:
```json
"variables": {
    "appServicePlanName": "[concat(parameters('authorName'),uniqueString(resourceGroup().id),'-asp')]",
    "webAppName": "[concat(parameters('authorName'),uniqueString(resourceGroup().id),'-as')]",
    "sqlServerName": "[concat(parameters('authorName'),uniqueString(resourceGroup().id),'-dbs')]"
}
```

With the tamplate I’m deploying three resources – App Service Plan, Web App and SQL Server:
```json
"resources": [
    {
        "comments": "Create the App Service Plan",
        "type": "Microsoft.Web/serverfarms",
        "apiVersion": "2018-02-01",
        "name": "[variables('appServicePlanName')]",
        "location": "[parameters('location')]",
        "sku": {
            "name": "S1"
        },
        "kind": "linux"
    },
    {
        "comments": "Create the Web App",
        "type": "Microsoft.Web/sites",
        "apiVersion": "2018-11-01",
        "name": "[variables('webAppName')]",
        "location": "[parameters('location')]",
        "dependsOn": [
            "[resourceId('Microsoft.Web/serverfarms', variables('appServicePlanName'))]"
        ],
        "kind": "app",
        "properties": {
            "enabled": true,
            "hostNameSslStates": [
                {
                    "name": "[concat(variables('webAppName'), '.azurewebsites.net')]",
                    "sslState": "Disabled",
                    "hostType": "Standard"
                },
                {
                    "name": "[concat(variables('webAppName'),'.scm.azurewebsites.net')]",
                    "sslState": "Disabled",
                    "hostType": "Repository"
                }
            ],
            "serverFarmId": "[resourceId('Microsoft.Web/serverfarms', variables('appServicePlanName'))]",
            "httpsOnly": true
        }
    },
    {
        "comments": "Create the logical SQL Server",
        "type": "Microsoft.Sql/servers",
        "apiVersion": "2019-06-01-preview",
        "name": "[variables('sqlServerName')]",
        "location": "[parameters('location')]",
        "kind": "v12.0",
        "properties": {
            "administratorLogin": "[parameters('authorName')]",
            "administratorLoginPassword": "[concat('P',uniqueString(variables('sqlServerName'),'!'))]",
            "version": "12.0",
            "publicNetworkAccess": "Enabled"
        }
    }
]
```

#### The challenge 
The possible outbound IP addresses for a Web App is a list of IP addresses, so we need to create some loop to go through the list and add those IP addresses into the SQL firewall. In ARM templates, this is possible using the [copy element](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/copy-resources). 
To get the object of the WebApp, we need to use the [reference function](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/template-functions-resource#reference). But the reference function has a limition – "you can't use the reference function to set the value of the count property in a copy loop". Just "great"!!!

After spending some time trying different solutions, the only possibility was to use linked templates.

#### Linked templates
To overcome the limitation of the reference function, we will create a new linked template, send the list of IP addresses as parameters to that template and add those IP addresses into the firewall using the linked template. First, we need to add several things into our main template.

Because of the way the linked templates works, you need to store them on a public location from where the Azure Resource Manager can access them. The Azure Storage Account is one of the options for saving the templates, either as publicly accessible blobs or private blobs available through the use of SAS tokens. For the production deployments, I strongly recommended you to use SAS tokens.

In the main template, we will create a new variable which at runtime will have a value of the full URL of the linked template.
```json
"addToSqlFirewallTemplateUrl": "[replace(deployment().properties.templateLink.uri, '/master.template.json', '/addtosqlfirewall.template.json')]"
```
Also in the main template, under resources, we are adding a definition for a new deployment that is using the linked template:
```json
{
    "comments": "In separate template (because of the reference nightmare!!!) configure the SQL firewall rules",
    "apiVersion": "2017-05-10",
    "name": "webAppIPsSQLFirewall",
    "type": "Microsoft.Resources/deployments",
    "properties": {
        "mode": "Incremental",
        "templateLink": {
            "uri": "[variables('addToSqlFirewallTemplateUrl')]",
            "contentVersion": "1.0.0.0"
        },
        "parameters": {
            "webAppOutboundIpAddresses": {
                "value": "[split(reference(concat('Microsoft.Web/sites/',variables('webAppName'))).possibleOutboundIpAddresses,',')]"
            },
            "sqlServerName": {
                "value": "[variables('sqlServerName')]"
            }
        }
    },
    "dependsOn": [
        "[resourceId('Microsoft.Sql/servers', variables('sqlServerName'))]",
        "[resourceId('Microsoft.Web/sites', variables('webAppName'))]"
    ]
}
```
As a parameter, we are sending the array of the possible outbound IP addresses of the WebApp and SQL Server name to the linked template.

The linked template, named addtosqlfirewall.template.json, looks like this:
```json
{
    "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "webAppOutboundIpAddresses": {
            "type": "array",
            "metadata": {
                "description": "Array of possible Outbound IP addresses for the Web Application"
            }
        },
        "sqlServerName": {
            "type": "string",
            "metadata": {
                "description": "Name of the logical SQL Server"
            }
        }
    },
    "resources": [
        {
            "comments": "Add the Outbound IP Addresses from the Web App",
            "type": "Microsoft.Sql/servers/firewallRules",
            "apiVersion": "2015-05-01-preview",
            "name": "[concat(parameters('sqlServerName'), '/Allow WebApp Outbound IP ',copyIndex('webAppOutboundIPAddressesCopy'))]",
            "properties": {
                "startIpAddress": "[parameters('webAppOutboundIpAddresses')[copyIndex('webAppOutboundIPAddressesCopy')]]",
                "endIpAddress": "[parameters('webAppOutboundIpAddresses')[copyIndex('webAppOutboundIPAddressesCopy')]]"
            },
            "copy": {
                "name": "webAppOutboundIPAddressesCopy",
                "count": "[length(parameters('webAppOutboundIpAddresses'))]"
            }
        }
    ]
}
```
#### Upload the templates to an Azure Storage Account
We will create a storage account and a container with publicly accessible blobs and upload the two templates to the container. The random string is needed because storage accounts are using globally unique names.
```bash
resourceGroup=<YOUR RESOURCE GROUP HERE>
randomString=$(cat /dev/urandom | tr -dc 'a-z' | fold -w 8 | head -n 1)
blobUrl=$(az storage account create -g $resourceGroup \
-n tosokr$randomString --query primaryEndpoints.blob -o tsv)
az storage container create --account-name tosokr$randomString \
--name templates --public-access blob
az storage copy -s "*" -d $blobUrl"templates"
```

#### Run the deployment
We will use Azure CLI to deploy the templates:
```bash
az group deployment create --name myDeployment \
--resource-group $resourceGroup --template-uri $blobUrl"templates/master.template.json"
```
Lets verify that the SQL firewall includes the outbound IP addresses from the web app:
```bash
sqlServerName=$(az sql server list \
--resource-group $resourceGroup --query [].name \
--o tsv | grep tosokr)
az sql server firewall-rule list \
-g $resourceGroup --server $sqlServerName \
--query '[].{RuleName:name,IPaddress:startIpAddress}' \
-o table
```
In my case, the output was:
```bash
RuleName                    IPaddress
--------------------------  ---------------
Allow WebApp Outbound IP 0  13.69.68.2
Allow WebApp Outbound IP 1  23.97.146.237
Allow WebApp Outbound IP 2  137.117.153.170
Allow WebApp Outbound IP 3  104.40.201.109
Allow WebApp Outbound IP 4  104.40.205.110
Allow WebApp Outbound IP 5  104.40.200.176
Allow WebApp Outbound IP 6  104.40.204.252
Allow WebApp Outbound IP 7  104.40.205.29
```
Grab a beer! Now you know how to use the reference function in a loop.