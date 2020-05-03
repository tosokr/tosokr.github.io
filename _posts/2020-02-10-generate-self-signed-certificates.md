---
title: Generate self signed certificates
date: 2020-02-14 21:00:00 +0000
description: How to generate self signed certificates including Root CA with openssl
categories: [Certificates]
tags: [Certificates]
header:
 teaser: "/assets/img/posts/teasers/certificates.png"
permalink: /posts/generate-self-signed-certificates/
---
In this post, I’m explaining how to generate a wildcard certificate for the custom domain with openssl, using a custom Certificate Authority. You can find the script in my Github repository [https://github.com/tosokr/Azure/blob/master/certificates/generateCertificates.sh](https://github.com/tosokr/Azure/blob/master/certificates/generateCertificates.sh)

1. Set the domain name
```shell
domainName="mycustomdomain.com"
```
2. Create the Root private key. Remove -des3 to create passwordless private key
```shell
openssl genrsa -des3 -out rootCA.key 4096
```
3. Create and self sign the Root Certificate
```shell
openssl req -x509 -new -nodes -key rootCA.key -sha256 -days 1024 -out rootCA.crt
```
4. Create the private certificate key
```shell
openssl genrsa -out $domainName.key 2048
```
5. Make sure that .rnd file is available
```shell
touch .rnd
```
6. Create the certificate signing requests
```shell
openssl req -new -sha256 -key $domainName.key -subj /CN=*.$domainName -out $domainName.csr 
```
7. Create the V3 extensions file
```shell
cat > v3.ext <<EOF
    authorityKeyIdentifier=keyid,issuer
    basicConstraints=CA:FALSE
    keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
    subjectAltName = @alt_names
    [alt_names]
    DNS.1 = *.$domainName
EOF
```
8. Generate the certificates using the mydomain csr and key along with the CA Root key
```shell
openssl x509 -req -in $domainName.csr -CA rootCA.crt -CAkey rootCA.key -CAcreateserial -out $domainName.crt -days 730 -sha256 -extfile v3.ext
```
9. Generate pfx file for the certificate
```shell
openssl pkcs12 -export -out $domainName.pfx -inkey $domainName.key -in $domainName.crt
```