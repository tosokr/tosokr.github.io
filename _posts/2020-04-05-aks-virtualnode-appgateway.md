---
title: Quickly and efficiently scale containerized applications using Azure Kubernetes Service, Container Instances, and Application Gateway 
date: 2020-04-06 22:00:00 +0000
description: How to deploy Azure Kubernetes Service with virtual nodes and Application Gateway ingress controller
categories: [Azure Kubernetes Service, Azure Container Instances, Application Gateway]
tags: [AKS,ACI,Application Gateway]
toc: false 
header:
 teaser: "/assets/img/posts/teasers/aks.png"
permalink: /posts/aks-virtualnode-appgateway/
---
Azure Kubernetes Service (AKS) is a managed Kubernetes cluster offering in Azure, meaning Microsoft is taking care of managing the Kubernetes masters. AKS is "free" – you only need to pay for the nodes (virtual machines). Because AKS is free, there is no coverage by guaranteed (financially-backed) SLA. Still, Microsoft is stating that "We will strive to attain at least 99.5% availability for the Kubernetes API server". For the Kubernetes nodes virtual machines, the SLA depends on the type of deployment: 99.99% for Availability Zones, 99.95% for Availability Set, and 99.9% for a single VM using Premium SSD or Ultra Disk.

Azure Container Instances (ACI) enables you to run containers on-demand, without managing or thinking about the infrastructure bellow.  This definition applies that it is a serverless container runtime offering. [Virtual Kubelet](https://github.com/virtual-kubelet/virtual-kubelet) is an open-source Kubernetes kubelet implementation that enables Kubernetes to talk with other APIs,  allowing the nodes to be backed by other services like ACI, AWS Fargate, IoT Edge, etc. From the AKS perspective, this means that instead of creating nodes using VM's and enable autoscaling using Virtual Machine Scale Sets,  we can use the virtual kubelet and schedule pods for execution on an ACI, give us "unlimited" scale capacity. These days, this is super easy to configure on the AKS, just enable the add-on named virtual-node and you are ready to go. Note: For running AKS, you need a minimum of one VM node.

Application Gateway (AppGateway) is a Layer 7 load balancer that can also act as an application firewall if you enable the Web Application Firewall module. Seating at the edge of the Virtual Network, it can do URL routing, SSL termination, end-to-end SSL, but still no support for mutual TLS. Depending on the amount of traffic that is hitting the  AppGateway, it can autoscale to handle the peak load. Application Gateway Ingress Controller (AGIC) is a Kubernetes application, which makes it possible for Azure Kubernetes Service (AKS) customers to leverage Azure's native Application Gateway L7 load-balancer to expose cloud software to the Internet. The main advantage of AGIC is that it enable AppGateway to talk to pods using their private IP directly and does not require NodePort or KubeProxy services, meaning better performances. You can find more info about AGIC [here](https://docs.microsoft.com/en-us/azure/application-gateway/ingress-controller-overview).

Using the commands below, we will create an AKS cluster, enable the virtual-nodes add-on on the cluster and deploy the AppGateway and AGIC. In the end, we will deploy a sample application to AKS, expose it to the Internet using AppGateway, generate a load on the app, and monitor pods auto-creation on the ACI.  To follow, you will need an Azure subscription and Azure CLI (you can use cloudshell for that matter – try to access the cloudshell using Windows Terminal, it works like a charm). What I did is I combined several examples that are out there into single deployment, of course, with some adjustments.  You can find the full script [here](https://raw.githubusercontent.com/tosokr/Azure/master/aks/aks-virtualnode-appgateway-deployment.sh). The end solution will look like this:

![Desktop View]({{ "/assets/img/posts/aks/AKS_ACI_AppGateway.jpg" | relative_url }})

Let's start.

####  1. Set the value of the variables we are going to use during the deployment
```shell
resourceGroupName="rg-aks"
location="westeurope"
aksClusterName="tosokr"
aksClusterNodeSize="Standard_DS1_v2"
aksClusterNodeCount=2
```
#### 2. Create the resource group
```shell
az group create --name $resourceGroupName --location $location
```
#### 3. Create the AKS cluster
Note: During the creation of the AKS cluster, if you choose to create a service principle automatically, you may face the following issue: 
>Operation failed with status: 'Bad Request'. Details: The credentials in ServicePrincipalProfile were invalid. Please see https://aka.ms/aks-sp-help for more details.

The cause of this issue is a data replication lag between the create call for the SP, confirmation, and then replication to the requested region. To avoid this, we will create the service principle manually:
```shell
servicePrinciplePassword=$(az ad sp create-for-rbac \
--skip-assignment --name myAKSClusterServicePrincipal \
--query password --output tsv) 
servicePrincipleId=$(az ad sp show --id http://myAKSClusterServicePrincipal \
--query appId --output tsv)
```
and will use that service principle when we create the AKS cluster:
```shell
az aks create --resource-group $resourceGroupName --name $aksClusterName \
--node-count $aksClusterNodeCount --location $location \
--generate-ssh-keys --service-principal $servicePrincipleId \
--client-secret $servicePrinciplePassword --node-vm-size $aksClusterNodeSize \
--network-plugin azure
```

#### 4. Enable the Virtual-node addon for the AKS, which will enable to use ACI for pods deployment. For this, we need to create a dedicated subnet in the Vnet where we deployed AKS. 
Note: When AKS is created, all connected resources such as vnet and node virtual machines are deployed in a separate resource group. Use that resource group just for storing resources with the same lifespan as the AKS cluster.
```shell
# get the AKS resource group for the nodes and vnet name
nodeResourceGroup=$(az aks show --resource-group rg-aks \
--name tosokr --query nodeResourceGroup --o tsv)
vnetName=$(az network vnet list --query [].name --o tsv \
--resource-group $nodeResourceGroup)
# create subnet for the ACI instances
az network vnet subnet create \
    --resource-group $nodeResourceGroup \
    --vnet-name $vnetName \
    --name aci-subnet \
    --address-prefixes 10.241.0.0/16
# enable virtual-node addon on the AKS cluster
az aks enable-addons --addons virtual-node \
--resource-group $resourceGroupName --name $aksClusterName \
--subnet-name aci-subnet
```
#### 5. Create the Application Gateway. We need to deploy the AppGateway into dedicated subnet in vnet where we deployed AKS
```shell
# create subnet for Application Gateway
az network vnet subnet create \
  --name ag-subnet \
  --resource-group $nodeResourceGroup \
  --vnet-name $vnetName \
  --address-prefix 10.242.0.0/27
# create public IP address for Application Gateway
az network public-ip create \
  --resource-group $nodeResourceGroup \
  --name myAGPublicIPAddress \
  --allocation-method Static \
  --sku Standard
# create the ApplicationGateway
az network application-gateway create \
  --name aksAppGateway \
  --location $location \
  --resource-group $nodeResourceGroup \
  --capacity 1 \
  --sku Standard_v2 \
  --http-settings-cookie-based-affinity Enabled \
  --public-ip-address myAGPublicIPAddress \
  --vnet-name $vnetName \
  --subnet ag-subnet
```
#### 6. Create [aad-pod-identity](https://github.com/Azure/aad-pod-identity) and identity for Azure Resource Manager operations. AAD Pod Identity enables Kubernetes applications to access cloud resources securely with Azure Active Directory.
```shell
# get the AKS credentials
az aks get-credentials --resource-group $resourceGroupName \
--name $aksClusterName
# get the subscriptionId
subscriptionId=$(az account show --query id -o tsv)
# create aad-pod-identity
kubectl apply -f \
https://raw.githubusercontent.com/Azure/aad-pod-identity/master/deploy/infra+/deployment-rbac.yaml
# create an Azure identity resource
azureIdentityId=$(az identity create -g $nodeResourceGroup -n azureIdentity \
--query id -o tsv)
azureIdentityClientId=$(az identity show --ids $azureIdentityId \
--query clientId -o tsv)
# install the Azure Identity into AKS
cat <<EOF | kubectl apply -f -
apiVersion: "aadpodidentity.k8s.io/v1"
kind: AzureIdentity
metadata:
  name: azureidentity
spec:
  type: 1
  ResourceID: $azureIdentityId
  ClientID: $azureIdentityClientId
EOF
# set the Azure Identity Binding
cat <<EOF | kubectl apply -f -
apiVersion: "aadpodidentity.k8s.io/v1"
kind: AzureIdentityBinding
metadata:
  name: azure-identity-binding
spec:
  AzureIdentity: azureidentity
  Selector: ingress-azure
EOF
# create an Azure identity and give it permissions to ARM
armAzureIdentityPrincipalId=$(az identity create -g $nodeResourceGroup \
-n armAzureIdentity --query principalId -o tsv)
# get the Application Gateway resourceid
appGatewayResourceId=$(az network application-gateway list \
--resource-group $nodeResourceGroup --query '[].id' -o tsv)
# get the nodeResourceGroup id
nodeResourceGroupId=$(az group show --name $nodeResourceGroup --query id -o tsv)
# give Contributor access to the Application Gateway
az role assignment create \
    --role Contributor \
    --assignee $armAzureIdentityPrincipalId \
    --scope $appGatewayResourceId
# give Reader access to the Resource Group
az role assignment create \
    --role Reader \
    --assignee $armAzureIdentityPrincipalId \
    --scope $nodeResourceGroupId
```
#### 7. Install the Application Gateway Ingress Controller into AKS
```shell
# install tiller for Helm v2
kubectl create serviceaccount \
--namespace kube-system tiller-sa
kubectl create clusterrolebinding tiller-cluster-rule \
--clusterrole=cluster-admin --serviceaccount=kube-system:tiller-sa
helm init --tiller-namespace kube-system --service-account tiller-sa
# add the application-gateway-kubernetes-ingress helm repo and perform a helm update
helm repo add application-gateway-kubernetes-ingress \
https://appgwingress.blob.core.windows.net/ingress-azure-helm-package/
helm repo update
# Install the Application Gateway Ingress Controler (AGIC) using helm
armAzureIdentityId=$(az identity show --name armAzureIdentity \
--resource-group $nodeResourceGroup --query id -o tsv)
armAzureIdentityClientId=$(az identity show --name armAzureIdentity \
--resource-group $nodeResourceGroup --query clientId -o tsv)
aksApiServerAddress=$(az aks show --resource-group rg-aks \
--name tosokr --query fqdn -o tsv)
helm install application-gateway-kubernetes-ingress/ingress-azure \
     --name ingress-azure \
     --namespace default \
     --debug \
     --set appgw.name=aksAppGateway \
     --set appgw.resourceGroup=$nodeResourceGroup \
     --set appgw.subscriptionId=$subscriptionId \
     --set appgw.shared=false \
     --set armAuth.type=aadPodIdentity \
     --set armAuth.identityResourceID=$armAzureIdentityId \
     --set armAuth.identityClientID=$armAzureIdentityClientId \
     --set rbac.enabled=true \
     --set verbosityLevel=3 \
     --set kubernetes.watchNamespace=default \
     --set aksClusterConfiguration.apiServerAddress=$aksApiServerAddress
```
After pod is created, view its details using:
```shell
kubectl describe pod -l app=ingress-azure
```
If you see the following messages, you need to manually edit the deployment (THIS IS AN AGIC BUG)
>Liveness probe failed: Get http://10.240.0.44:8123/health/alive: dial tcp 10.240.0.44:8123: connect: connection refused
Readiness probe failed: Get http://10.240.0.44:8123/health/ready: dial tcp 10.240.0.44:8123: connect: connection refused
```shell
kubectl edit deployment ingress-azure
```
and remove the livenessProbe and readinessProbe sections from the yaml file

#### 8. Let's deploy a simple demo application into the AKS. 
We will deploy the pods on the virtual-kubelet (see the nodeSelector and tolerations definitions below), create a horizontal pod autoscaler for the deployment with a maximum of 3 pods, service and ingress controller for exposing the pods to the Internet.
```shell
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: aci-aspnetapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: aci-aspnetapp
  template:
    metadata:
      labels:
        app: aci-aspnetapp
    spec:
      containers:
      - image: "mcr.microsoft.com/dotnet/core/samples:aspnetapp"
        name: aspnetapp-image
        ports:
        - containerPort: 80
          protocol: TCP
        resources:
          requests:
            cpu: 250m
            memory: 250Mi
          limits:
            cpu: 250m
            memory: 250Mi
      nodeSelector:
        kubernetes.io/role: agent
        beta.kubernetes.io/os: linux
        type: virtual-kubelet
      tolerations:
      - key: virtual-kubelet.io/provider
        operator: Exists
      - key: azure.com/aci
        effect: NoSchedule
      
---

apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: aci-aspnetapp-hpa
spec:
  maxReplicas: 3 # define max replica count
  minReplicas: 1  # define min replica count
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: aci-aspnetapp
  targetCPUUtilizationPercentage: 50 # target CPU utilization

---

apiVersion: v1
kind: Service
metadata:
  name: aci-aspnetapp
spec:
  selector:
    app: aci-aspnetapp
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80

---

apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: aci-aspnetapp
  annotations:
    kubernetes.io/ingress.class: azure/application-gateway
spec:
  rules:
  - http:
      paths:
      - path: /
        backend:
          serviceName: aci-aspnetapp
          servicePort: 80
EOF
```
#### 9. Generate some traffic to the application
```shell
# you need to install go first: sudo apt install golang-go 
export GOPATH=~/go
export PATH=$GOPATH/bin:$PATH
go get -u github.com/rakyll/hey
hey -z 20m http://<whatever-the-ingress-url-is>
```
#### 10. In a new terminal, view how pods are autoscaling on Azure Container Instances
```shell
kubectl get hpa aci-aspnetapp-hpa -w
```
#### Grab a beer. You did it!