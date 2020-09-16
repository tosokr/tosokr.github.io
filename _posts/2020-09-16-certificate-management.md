---
title: Certificate management using Let's Encrypt certificates and Azure Key Vault
date: 2020-09-16 00:00:00 +0000
description: Certificate management using Let's Encrypt certificates and Azure Key Vault is a cost-efficient solution that you can use for securing your services
categories: [Key Vault]
tags: [Key Vault,Certificates]
header:
 teaser: "/assets/img/posts/teasers/certificates.png"
permalink: /keyvault/certificate-management/
excerpt: Let's Encrypt is a free, automated, and open certificate authority (CA), run for the public's benefit. Key Vault ACMEBot is an open-source solution for automatic certificate management that allows you to quickly request a certificate, store it into a Key Vault, and automatically renew it before it expires.
---
### The Let's Encrypt project
Let's Encrypt is a free, automated, and open certificate authority (CA), run for the public's benefit. It is a service provided by the Internet Security Research Group (ISRG). ISRG was founded in May of 2013 to serve as a home for public-benefit digital infrastructure projects, the first of which was the Let's Encrypt certificate authority. The organization is sponsored by companies like Cisco, Facebook, Google, Red Hat, Internet Society, and much more. 

Let's Encrypt certificates are trusted out of the box on almost all [modern browsers and operating systems](https://letsencrypt.org/docs/certificate-compatibility/). Until today, Let's Encrypt have issued over one billion certificates, issuing between 1.5-2.0 million certificates per day. The statistics are available on the following [page](https://letsencrypt.org/stats/)
Let's Encrypt issues certificates using the Automatic Certificate Management Environment (ACME) protocol. The protocol allows issuing a certificate without human intervention. There are two steps to this process:
1.	Domain ownership verification 
2.	Request, renew, and revoke certificates for that domain.

More info on how Let's Encrypt issues certificates is available on the following [link](https://letsencrypt.org/how-it-works/)

There are several concerns about using Let's Encrypt for issuing certificates for production enterprise workloads. Please, take consideration of the following before deciding to proceed with Let's Encrypt:
1.	Let's Encrypt is run by a small team and relies on automation to keep costs down. They are not offering any direct support, meaning no SLA (because service is free).
2.	Certificates are standard Domain Validation certificates, so you can use them for any server that uses a domain name, like web servers, mail servers, FTP servers, and many more. Email encryption and code signing require a different type of certificate that Let's Encrypt do not issue.
3.	The validity of the certificates is 90 days. 
4.	Let's Encrypt provides rate limits to ensure fair usage by as many people as possible. The main limit is Certificates per Registered Domain (50 per week), excluding the renewals. There is a possibility to increase this rate limit by contacting the Let's Encrypt team. The rate limits and process for the increase is documented [here](https://letsencrypt.org/docs/rate-limits/)
5.	You can eventually bypass the rate limits without rate limit increase by the Let's Encrypt team using multi-domain (SAN) Certificates (up to 100 names per certificate). By adding multiple domains, the certificate size increases, which can slow down the HTTPS handshakes.

### Solution for automatic certificate management
[Key Vault ACMEbot](https://github.com/shibayan/keyvault-acmebot) is an open-source solution developed by Tatsuro Shibamura, a Microsoft MVP for Azure. On his [blog](https://samcogan.com/lets-encrypt-certificates-in-azure-with-acmebotot/), Sam Cogan did an excellent overview of the solution. I highly recommend checking that before proceeding with the installation.

ACMEBot runs as an Azure Function using the consumption plan, so you only pay for the executions (and get a hefty free allowance). Alongside the function, a storage account (needed for the function) and app insights instance (for function's observability) are part of the deployment. The private keys for the certificates are generated directly into the Key Vault (the private key never leaves), where also the issued certificates are imported. For WEB/API authentication, you can enable App Service Auth on the function level and integrate it with the Azure Active Directory, meaning only accounts from your tenant can log in. The integration between the different components composing this solution is with the use of Managed Identities.

To request a new certificate, you need to call the Azure Functions "add-certificate" function, through a browser or an API call. After choosing the domain and hostname for the certificate, several other functions are used to call out and undertake the issuing process. These hit the following resources:
1.	Azure DNS - ACMEBot uses the DNS validation process to verify ownership of the domain you want to issue the certificate for. To do this, the function needs to be able to create and delete records in the Azure DNS zone that hosts your domain. 
2.	Let's Encrypt - Once the DNS record is set up, calls are made to the Let's encrypt API to generate and download the certificate
3.	Azure KeyVault - once the certificate is created it is stored in Azure Key Vault

This process runs when you create a new certificate. What ACMEBot also does is handling certificate renewals. It will detect certificates that are reaching expiry in the next 30 days and call out to Let's Encrypt to renew them and place the new certificate into Key Vault.
![Desktop View]({{ "/assets/img/posts/keyvault/certificateManagement.png" | relative_url }})

The recommended deployment model for the solution is one instance per workload (application) per environment, part of the infrastructure CI/CD pipeline. 

The contents of the Key Vault are automatically replicated within the region and to a secondary region at least 150 miles away but within the same geography to maintain high durability of the certificates. In the event of a region failover, it may take a few minutes for the service to failover. Requests that are made during this time before failover may fail. Also, during failover, the Key Vault is in read-only mode.
In the case of Let's Encrypt failure, you will not be able to request new certificates, and also the Certificate Revocation List (CRL) may not be accessible. By default, the issued certificates are renewed 30 days before the expiration date, meaning there is a minimal risk that the certificate will expire without being able to renew.

If the proposed solution looks feasible for you, and you want to make it enterprise ready, you will probably need to:
1.	Clone the code, create appropriate CI/CD, and make it enterprise production-ready. Also, monitor the developments of the ACME protocol and incorporate any new versions 
2.	Implement authorization on top of the provided Azure AD authentication
3.	Add certificate revocation capability
4.	Improve the monitoring and alerting

