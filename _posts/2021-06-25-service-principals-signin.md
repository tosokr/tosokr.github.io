---
title: Get login details for service principals
date: 2021-06-26 00:00:00 +0000
description: Microsoft added support for querying audit reports for service principals in the beta version of the Microsoft Graph APIs. This functionality can help you build a governance workflow for the service principals because you can access information like signIn time, IP address, and location. Learn how to do that because it is not well documented in the official documentation. 
categories: [Security]
tags: [MicrosoftGraph,ServicePrincipals]
toc: false 
header:
 teaser: "/assets/img/posts/teasers/resourceGraphQueries.png"
permalink: /security/service-principals-signin-activities/
excerpt: Microsoft added support for querying audit reports for service principals in the beta version of the Microsoft Graph APIs. This functionality can help you build a governance workflow for the service principals because you can access information like signIn time, IP address, and location. Learn how to do that because it is not well documented in the official documentation.  
---
You can use Microsoft Graph APIs to access Azure AD activity report. In version 1.0, you can access those reports only for user principals. However, in the beta version of the APIs, Microsoft added support for doing the same for service principals. This functionality can help you build a governance workflow for the service principals because you can access information like signIn time, IP address, and location. When creating that governance workflow, one challenge to consider is that that information is kept in Azure Active Directory only for 30 days. I recommend using Logic Apps for building such workflow. Check the [following post](https://aztoso.com/security/microsoft-graph-permissions-managed-identity/) on adding Microsoft Graph API permissions to a Managed Identity. To be able to query the Microsoft Graph API, you need the AuditLog.Read.All permission.

To get all signIn activities for a specified service principal in the last 30 days (if any):
*YOUR_APP_ID* - Application Id of the service principal.
```http 
GET https://graph.microsoft.com/beta/auditLogs/signIns?$filter=signInEventTypes/any(t: t eq 'servicePrincipal') and (appId eq 'YOUR_APP_ID')
```

To get the last signIn activity for a specified service principal in the last 30 days (if any):
```http
GET https://graph.microsoft.com/beta/auditLogs/signIns?$filter=signInEventTypes/any(t: t eq 'servicePrincipal') and (appId eq 'YOUR_APP_ID')&top=1&orderby=createdDateTime%20desc
```
I hope this will save you some valuable time trying to figure out how to call the API because it is not well documented in the official documentation.