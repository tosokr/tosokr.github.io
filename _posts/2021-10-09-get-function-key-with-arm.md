---
title: Azure Function keys - what are those and how to access them
date: 2021-10-09 00:00:00 +0000
description: Azure Function Keys are used for authorizing access to the functions. The host and the master key exist at the Function App level, while each function also has a function-specific key that can be used to access that function. You can access the keys from ARM templates, in the portal or using Azure CLI. 
categories: [Functions]
tags: [Functions,ARM]
toc: true 
header:
 teaser: "/assets/img/posts/teasers/functions.png"
permalink: /functions/keys/
excerpt: Azure Function Keys are used for authorizing access to the functions. The host and the master key exist at the Function App level, while each function also has a function-specific key that can be used to access that function. You can access the keys from ARM templates, in the portal or using Azure CLI.
---
# Usage of keys in Azure Functions

Azure Functions offer three levels of authorization:

* anonymous: no key is required
* function: a function-specific or a host key is required
* admin: the master key is required

Function access keys are the simple way to protect an Azure Function from unauthorized access. You can provide the access key in the _code_ query parameter or in the _x-functions-key_ HTTP header when accessing a function. There are two access scopes for function-level keys:

* function: these keys apply only to the specific functions under which they are defined
* host: keys with a host scope can be used to access all functions within the function app

The master key also provides access to all functions within the function app and administrative access to the runtime REST APIs. As a best practice, never use the master key in client applications to access an Azure Function.

One particular type of key, named system key, is used by extensions installed in the Azure Function, such as Event Grid and Durable Functions.

# How to get the keys value?

In the examples below, the following applies:

* _rg-functions-test_ - resource group where the function app is deployed
* _aztosotest_ - the name of the function app
* _Hello_ - the name of the function

## Azure Portal

After opening the Function App in the portal, you can view the host and system keys by selecting _App keys_ from the left sidebar:

![Desktop View]({{ "/assets/img/posts/function/function-host_and_system_keys.png" | relative_url }})

To view the function-specific keys, open the function and chose _Function Keys_ from the left sidebar:

![Desktop View]({{ "/assets/img/posts/function/function-function_level_keys.png" | relative_url }})

## ARM template

Note: if the function is deployed in a different resource group, you need to specify the resource group name as a first parameter in the [resourceId](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/template-functions-resource#resourceid) function


Host(default) key:

```json
listkeys(concat(resourceId('Microsoft.Web/sites', 'aztosotest'), '/host/default/'),'2021-02-01').functionKeys.default
```

Master key:

```json
listkeys(concat(resourceId('Microsoft.Web/sites', 'aztosotest'), '/host/default/'),'2021-02-01').masterKey
```

System key (named durabletask_extension):

```json
listkeys(concat(resourceId('Microsoft.Web/sites', 'aztosotest'), '/host/default/'),'2021-02-01').systemkeys.durabletask_extension
```

Function-specific key:

```json
listKeys(resourceId('Microsoft.Web/sites/functions', 'aztosotest', 'Hello'),'2021-02-01').default
```

## Azure CLI

To get the host, master, and system key, use the following command:

```bash
az functionapp keys list -g rg-functions-test -n aztosotest
```
```json
{
  "functionKeys": {
    "default": "68ahyfSAT2PF5HfIuGMyc8Bv6NErsFxnk6lT0l2fYacYCnVameOAmw=="
  },
  "masterKey": "N4O1a4ng/gLemQEiDFF9HUpNznT/pFGNSnTj4jQXVOhvZAa8MivuOg==",
  "systemKeys": {
    "durabletask_extension": "DtieTxAOV5x76Zfv6NWBqbbW3/u4YiHVhycZfPgn2VIgkcPAGP8kaA=="
  }
}
```

The following command will list the function-specific keys:

```bash
az functionapp function keys list -g rg-functions-test -n aztosotest --function-name Hello
```
```json
{
  "default": "ukRsq3yoeayz5ELYgTGK3fbU3yfbHXavy5m9Y6ci0ZSykqFYUxyd3Q==",
  "id": null,
  "kind": null,
  "name": null,
  "properties": null,
  "systemData": null,
  "type": null
}
```