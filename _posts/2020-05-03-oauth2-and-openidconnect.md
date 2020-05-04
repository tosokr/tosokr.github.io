---
title: Back to basics - OAuth 2.0 and OpenID Connect   
date: 2020-05-03 12:00:00 +0000
description: Description of OAuth 2.0 and OpenID Connect, best practices and when to use each
categories: [Security]
tags: [OpenID,OAuth2]
header:
 teaser: "/assets/img/posts/teasers/auth.png"
permalink: /security/oauth2-and-openidconnect/
excerpt: OAuth 2.0 is the industry-standard protocol for authorization. OAuth 2.0 focuses on client developer simplicity while providing specific authorization flows for web applications, desktop applications, mobile phones, and living room devices. OpenID Connect is a simple identity layer built on top of the OAuth 2.0 protocol. It includes information about the end-user in the form of an id_token that verifies the identity of the user and provides basic profile information about the user.
---
### What is OAuth 2.0 and OpenID Connect?
>OAuth 2.0 is the industry-standard protocol for authorization. OAuth 2.0 focuses on client developer simplicity while providing specific authorization flows for web applications, desktop applications, mobile phones, and living room devices. - [oauth.net](https://oauth.net/2/)

Because OAuth 2.0 is an authorization protocol, it leads to the following problems when used for authentication: 
- No standard way to get the user's information
- Every implementation is a little different
- No common set of scope

OpenID Connect is a simple identity layer built on top of the OAuth 2.0 protocol. OAuth 2.0 defines mechanisms to obtain and use access tokens to access protected resources, but it does not specify standard methods to provide identity information. OpenID Connect implements authentication as an extension to the OAuth 2.0 authorization process. It includes information about the end-user in the form of an id_token that verifies the identity of the user and provides basic profile information about the user.

The OpenID Connect flow look like this:

![Desktop View]({{ "/assets/img/posts/aad/OpenIDFlow.png" | relative_url }})

1. A user running browser signs in, enter credentials and consents to permissions.
2. The /oauth2/authorize endpoint returns id_token an authorization code to the browser.
3. The browser redirects to redirect URI (the webserver).
4. The web server validates id_token and sets a session cookie.
5. The web server requests an OAuth bearer token from the /oauth2/token endpoint and provides the authorization code.
6. The /oauth2/token endpoint returns an access token and a refresh_token.
7. The web server calls a web API with a token in the authorization header.
8. The web API validates the token.
9. The web API returns secure data to the webserver.

In OAuth 2.0, the term "grant type" refers to the way an application gets an access token. Each grant type is optimized for a particular use case. Todays best practices for usage of the grant types are: 

- Web application with server backend: authorization code flow
- Native or mobile app: authorization code with PKCE flow
- JavaScript app (Single Page Application) with API backend:
  - authorization code with PKCE flow (if you can)
  - implicit flow (if you must)
- Microservices and APIs: client credentials flow

### OAuth 2.0 authorization code flow 
OAuth 2.0 authorization code grant uses two separate endpoints. The authorization endpoint is used for the user interaction phase, which results in an authorization code. The token endpoint is then used by the client for exchanging the code for an access token, and often a refresh token as well. Web applications are required to present their application credentials to the token endpoint so that the authorization server can authenticate the client. To authorize access to Azure AD web applications by using the OAuth 2.0 code grant flow, you need to:

1. Register your application with your Azure AD tenant (this provides an Application ID for the application, as well as enable it to receive tokens)
2. Request an authorization code (the authorization code flow begins with the client directing the user to the /authorize endpoint. The client indicates the permissions it needs to acquire from the user)
3. Use the authorization code to request an access token (include the client secret into the request, for security verification)
4. Use the access token to access the resource (add the Authorization: Bearer header in the requests)
5. Refresh the access token

The authorization code flow is illustrated in the picture above, for the Open ID Connect flow. 

### OAuth 2.0 client credentials flow 
OAuth 2.0 Client Credentials Grant Flow permits a web service (serving the role of a confidential client) to use its credentials to authenticate when calling another web service instead of impersonating a user. In this scenario, the client is typically a middle-tier web service, a daemon service, or a website. For a higher level of assurance, Azure AD also allows the calling service to use a certificate (instead of a shared secret) as a credential.

The picture illustrates how the client credentials grant flow works in Azure AD.

![Desktop View]({{ "/assets/img/posts/aad/clientCredentialsFlow.png" | relative_url }})

1. The client application authenticates to the Azure AD token issuance endpoint and requests an access token.
2. The Azure AD token issuance endpoint issues the access token.
3. The access token is used to authenticate to the secured resource.
4. Data from the secured resource is returned to the client application.

### Summary
Use OpenID Connect for (Authentication):
- Logging the user in
- Making your accounts available in other systems

Use OAuth 2.0 for (Authorization):
- Granting access to your API
- Getting access to user data in other systems

