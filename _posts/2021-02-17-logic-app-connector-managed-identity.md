---
title: Logic App connector for Key Vault using Managed Identity
date: 2021-02-17 00:00:00 +0000
description: Connectors provide quick access from Azure Logic Apps to events, data, and actions across other apps, services and platforms. One of the frequently used connectors in Logic Apps is the one for connecting to the Azure Key Vault resource. Today when you create a Key Vault connection in the portal, you can choose "Connect with managed identity". Learn how to do the same using an ARM template.
categories: [Logic App]
tags: [ARM,Logic App, Key Vault]
toc: false 
header:
 teaser: "/assets/img/posts/teasers/logicApps.png"
permalink: /logic-app/keyvault-connector-with-managed-identity/
excerpt: Connectors provide quick access from Azure Logic Apps to events, data, and actions across other apps, services and platforms. One of the frequently used connectors is the one for connecting to the Azure Key Vault resource. Today when you create a Key Vault connection in the portal, you can choose "Connect with managed identity". Learn how to do the same using an ARM template.
---
Connectors provide quick access from Azure Logic Apps to events, data, and actions across other apps, services, systems, protocols, and platforms. One of the frequently used connectors is the one for connecting to the Azure Key Vault resource.

Before introducing support for using a Managed Identity with the Azure Key Vault connector, if we wanted to use the Logic App's identity to access the Key Vault, we needed to use the HTTP action. Today when you create a Key Vault connection, you can choose "Connect with managed identity".

![Desktop View]({{ "/assets/img/posts/logicApp/logicAppKeyVaultMI.png" | relative_url }})

If you deploy your Logic Apps with automation, you can't find any information in the documentation on how to do the same thing with an ARM template. Use the following ARM snippet to create a connection to the Key Vault using Managed Identity:
```json
 {
    "type": "Microsoft.Web/connections",
    "apiVersion": "2016-06-01",
    "name": "keyvault",
    "location": "[parameters('location')]",
    "kind": "V1",
    "properties": {
        "displayName": "[parameters('keyVaultName')]",
        "parameterValueType": "Alternative",
        "alternativeParameterValues": {
            "vaultName": "[parameters('keyVaultName')]"
        },
        "customParameterValues": {},
        "api": {
            "id": "[subscriptionResourceId('Microsoft.Web/locations/managedApis', parameters('location'), 'keyvault')]"
        }
    }
}
```

In the Logic App definition, the definition of the connection needs to include the ManagedServiceIdentity authentication:
```json
"$connections": {
    "value": {
        "keyvault": {
            "connectionId": "[resourceId('Microsoft.Web/connections', 'keyvault')]",
            "connectionName": "keyvault",
            "connectionProperties": {
                "authentication": {
                    "type": "ManagedServiceIdentity"
                }
            },
            "id": "[subscriptionResourceId('Microsoft.Web/locations/managedApis', parameters('location'), 'keyvault')]"
        }
    }
}
```
