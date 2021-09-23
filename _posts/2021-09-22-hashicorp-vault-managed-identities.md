---
title: Access HashiCorp Vault secrets from AKS using Managed Identities
date: 2021-09-22 00:00:00 +0000
description: HashiCorp Vault agent and the CSI (Container Storage Interface) provider use Kubernetes type of authentication, based on Kubernetes Service Account Token. Azure type of authentication, with the use of Managed Identities, is supported by the API endpoint. Init scripts are one of the options to add that support in the AKS. 
categories: [Azure Kubernetes Service]
tags: [AKS]
toc: true 
header:
 teaser: "/assets/img/posts/teasers/hcv.png"
permalink: /aks/hcv-secrets-managed-identities/
excerpt: HashiCorp Vault agent and the CSI (Container Storage Interface) provider use Kubernetes type of authentication, based on Kubernetes Service Account Token. Azure type of authentication, with the use of Managed Identities, is supported by the API endpoint. Init scripts are one of the options to add that support in the AKS. 
---

# Overview

Two options exist for integrating HashiCorp Vault secrets out-of-the-box into the Kubernetes cluster: with the HashiCorp Vault agent or using the CSI (Container Storage Interface) provider. CSI is a Kubernetes native way of integrating external storage and secret management solutions (through the CSI Secrets store extension), and it is a recommended way to implement. The drawback of using those two options is the authentication part to the Vault. Both use kubernetes type of authentication, based on Kubernetes Service Account Token. Azure type of authentication (with aad-pod-identity) is not supported.

On the other hand, the HashiCorp Vault API supports different types of authentication, including Azure. It means that for API calls, you can use a bearer (JWT) token from a Managed Identity. The drawback is that you need to add the secret management part into your application stack or use an init container.

# Managed Identities with init containers

A Pod can have multiple containers running apps within it, but it can also have one or more init containers run before the app containers are started. Init containers are precisely like regular containers, except:

* Init containers always run to completion.
* Each init container must complete successfully before the next one starts.

Init containers offer a mechanism to block or delay app container startup until a set of preconditions are met. One of those conditions can be a successful fetch of a secret from the HashiCorp Vault. As HashiCorp Vault API endpoint supports Azure authentication, if the pod has assigned pod identity, the init container can assume that identity and use it to get a bearer token to authenticate to HCV. The following is an example of such pod definition:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hcv-demo
  namespace: hcv-test
  labels:
    aadpodidbinding: id-hcv-client-identity
spec:
  containers:
  - name: demo
    image: nginx
    imagePullPolicy: Always
    volumeMounts:
      - mountPath: /mnt/secrets/hcv
        name: secrets
  initContainers:
  - name: demo-init
    image: bitnami/minideb:latest
    env:
    - name: VAULT_ADDR
      value: "<VAULT_ADDRESS>"
    - name: VAULT_NAMESPACE
      value: "<VAULT_NAMESPACE>"
    - name: VAULT_ROLE
      value: "<VAULT_ROLE>"
    - name : SECRET_PATH
      value: "supersecrets/data/secret1"
    - name: MOUNT_PATH
      value: "/secrets/supersecret1"
    command: ["bin/bash", "-c"]
    args: [
      "apt update && apt install jq curl -y;
      metadata=$(curl -H Metadata:true 'http://169.254.169.254/metadata/instance?api-version=2019-08-15');
      subscription_id=$(echo $metadata | jq -r .compute.subscriptionId);
      vm_name=$(echo $metadata | jq -r .compute.name);
      vmss_name=$(echo $metadata | jq -r .compute.vmScaleSetName);
      resource_group_name=$(echo $metadata | jq -r .compute.resourceGroupName);
      jwt=$(curl 'http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https%3A%2F%2Fmanagement.azure.com%2F' -H Metadata:true -s | jq -r .access_token);
      echo '{\"role\": \"'$VAULT_ROLE'\",\"jwt\": \"'$jwt'\",\"subscription_id\": \"'$subscription_id'\",\"resource_group_name\": \"'$resource_group_name'\",\"vm_name\": \"'$vm_name'\",\"vmss_name\": \"'$vmss_name'\"}' > auth_payload_complete.json;
      token=$(curl --request POST -H \"X-Vault-Namespace: $VAULT_NAMESPACE\" --data @auth_payload_complete.json ${VAULT_ADDR}/v1/auth/azure/login | jq -r .auth.client_token);
      secret=$(curl -H \"X-Vault-Token: ${token}\" -H \"X-Vault-Namespace: $VAULT_NAMESPACE\" ${VAULT_ADDR}/v1/${SECRET_PATH} | jq -r .data.data.password);
      echo $secret > $MOUNT_PATH;
      if [ \"$secret\" == \"\" ] || [ \"$secret\" == \"null\" ]; then
        echo \"Failed to get the secret\";
        exit 1;
      fi
     "
    ]
    volumeMounts:
    - mountPath: /secrets
      name: secrets
  volumes:
    - name: secrets
      emptyDir: {}
```

The init script in the above example will use the _id-hcv-client-identity_ AzureIdentity (from aad-pod-identity) to grab the HashiCorp Vault secret with path _supersecrets/data/secret1_ and save it as _/secrets/supersecret1_. The application pod (nginx in the above example) will have the secret mounted as _/mnt/secrets/hcv/supersecret1_.