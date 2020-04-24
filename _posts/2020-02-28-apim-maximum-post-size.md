---
title: Limit the size of POST requests in Azure API Management
date: 2020-02-28 13:00:00 +0000
description: Azure API Management maximum size requests post
categories: [API Management]
tags: [APIMPolicy]
header:
 teaser: "/assets/img/posts/teasers/appConfiguration.png"
---
If there is a need for a file upload support using API POST request, and there is a maximum file size set the backend, the best place to validate the file size is at the APIM.
The policy definition bellow, checks the request method and the size of the request. If the HTTP method is POST and content size is above 50MB, the API will immediately return status code 413 “Payload Too Large” to the caller. Otherwise, it will continue processing other policy definitions and eventually hit the backend. 
```xml
<inbound>
	<base />
	<choose>
		<when condition="@(context.Request.Method == "POST")">
			<set-variable name="bodySize" value="@(context.Request.Headers["Content-Length"][0])" />
			<choose>
				<when condition="@(int.Parse(context.Variables.GetValueOrDefault<string>("bodySize"))<52428800)">
					<!--let it pass through by doing nothing-->
				</when>
				<otherwise>
					<return-response>
						<set-status code="413" reason="Payload Too Large" />
						<set-body>@{
								return "Maximum allowed size for the POST requests is 52428800 bytes (50 MB). This request has size of "+ context.Variables.GetValueOrDefault<string>("bodySize") +" bytes";
							} 
						</set-body>
					</return-response>
				</otherwise>
			</choose>
		</when>
	</choose>
</inbound>
```