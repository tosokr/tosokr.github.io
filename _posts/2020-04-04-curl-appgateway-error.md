---
title: SSL certificate error when accessing Application Gateway listener using curl
date: 2020-04-04 19:00:00 +0000
description: curl (60) SSL certificate problem unable to get local issuer certificate
categories: [Application Gateway]
tags: [Application Gateway,Certificates]
header:
 teaser: "/assets/img/posts/teasers/appGateway.png"
permalink: /posts/curl-appgateway-error/
---
If you get the following error when you try to open a webpage using Linux command-line tool curl:
>curl: (60) SSL certificate problem: unable to get local issuer certificate
More details here: https://curl.haxx.se/docs/sslcerts.html

it means that curl doesn't have the root certificate to verify the website certificate. To fix this issue, you will need the .pfx certificate, the password for the .pfx certificate, and the certificate chain (root and/or intermediate certificate). The solution is to include the certificate chain into the .pfx file and set that certificate on the https listener in Application Gateway. Let's start.

1.	Extract the key from the .pfx certificate. When prompted, enter the password
```shell
openssl pkcs12 -in myOldCertificate.pfx -nocerts -out private.key
```
2.	Extract the certificate data from the .pfx
```shell
openssl pkcs12 -in myOldCertificate.pfx -clcerts -nokeys -out certificate.crt
```
3.	Create a certificate chain using the root and intermediate certificates
```shell
cat root.crt > cert-chain.crt
cat intermediate.crt >> cert-chain.crt
```
4.	Create the new .pfx certificate
```shell
openssl pkcs12 -export -out myNewCertificate.pfx -inkey private.key -in certificate.crt -certfile cert-chain.crt
```
5.	Add the new pfx to the https listener in Application Gateway (or upload it to Key Vault)

I hope this will save you couple of hours debugging. Cheers!
