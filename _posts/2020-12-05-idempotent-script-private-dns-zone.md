---
title: Idempotent script to provision Private DNS Zone in Azure
date: 2020-12-05 00:00:00 +0000
description: Following industry standards and terms, the Azure Well-Architected Framework provides a set of Azure architecture best practices that support your cloud solution success.
categories: [Miscellaneous]
tags: [Private DNS Zones]
header:
 teaser: "/assets/img/posts/teasers/privateDNSZones.png"
permalink: /misc/provision-private-dns-zone/
excerpt: Azure Private DNS provides a reliable, secure DNS service to manage and resolve domain names in a virtual network. By using private DNS zones, you can use custom domain names rather than the Azure-provided names. The private DNS zone is reachable from all virtual networks linked to that zone. If, for some reason, you can't use an ARM template or Terraform, but you still want your deployments to be idempotent, you have an option to create idempotent deployments using scripts.
---
Azure Private DNS provides a reliable, secure DNS service to manage and resolve domain names in a virtual network. By using private DNS zones, you can use custom domain names rather than the Azure-provided names. The private DNS zone is reachable from all virtual networks linked to that zone.

If you can't use an ARM template or Terraform for some reason, but you still want your deployments to be idempotent, you have an option to create idempotent deployments using scripts. For one such scenario, I made a bash script using Azure CLI to provision the Private DNS Zone resource in Azure.
The script accepts named arguments. The arguments not explicitly listed in the while loop are treated as VNET links.    
```bash
./createPrivateDNSZone.sh --dns-zone-name <zone_name> --resource-group <resource_group> \
--location "<location>" --subscription_id <subscription> --tenant_id <tenant_id> \
--client_id <service_principal_id> --client_secret <service_pricipal_password> \
--<name_of_the_vnet_link> "<vnet_id>" --<name_of_the_vnet_link_2> "<vnet_id>"
```

Here is the content of the script. If you are not using the service principal, just remove the login part from the script. 
```bash
VNET_LINKS=()

while [[ $# -gt 0 ]]
do
key="$1"
    case $key in
        --subscription_id)
        SUBSCRIPTION_ID="$2"
        shift # past argument
        shift # past value
        ;;
        --resource-group)
        RESOURCE_GROUP="$2"
        shift # past argument
        shift # past value
        ;;
        --dns-zone-name)
        DNS_ZONE_NAME="$2"
        shift # past argument
        shift # past value
        ;;   
        --client_id)
        CLIENT_ID="$2"
        shift # past argument
        shift # past value
        ;;
        --client_secret)
        CLIENT_SECRET="$2"
        shift # past argument
        shift # past value
        ;;
        --tenant_id)
        TENANT_ID="$2"
        shift # past argument
        shift # past value
        ;;
        --location)
        LOCATION="$2"
        shift # past argument
        shift # past value
        ;;
        *)  
        VNET_LINKS+=("$1") # save it in an array for later
        shift # past argument    
        ;;
    esac
done

# LOGIN TO THE SUBSCRIPTION
az login --service-principal --username $CLIENT_ID --password $CLIENT_SECRET \
--tenant $TENANT_ID --output none
az account set --subscription $SUBSCRIPTION_ID

# CREATE THE RESOURCE GROUP
resourceGroupExists=$(az group exists --name "$RESOURCE_GROUP")
if [ "$resourceGroupExists" == "false" ]; then 
    echo "Creating resource group: "$RESOURCE_GROUP" in location: ""$LOCATION"
    az group create --name "$RESOURCE_GROUP" --location "$LOCATION"
fi

# CREATE THE PRIVATE DNS ZONE
privateDNSZoneExists=$(az network private-dns zone list -g "$RESOURCE_GROUP" \
--query "[?name=='$DNS_ZONE_NAME'].name" -o tsv)
if [ "$privateDNSZoneExists" != "$DNS_ZONE_NAME" ]; then
    echo "Creating Private DNS Zone: "$DNS_ZONE_NAME" in resource group: "$RESOURCE_GROUP
    az network private-dns zone create -g $RESOURCE_GROUP -n $DNS_ZONE_NAME
fi

# CREATE THE VNET LINKS
i=0 
while [[ i -lt ${#VNET_LINKS[@]} ]]
do    
    LINK_NAME="${VNET_LINKS[$i]/--/}"
    VNET_ID="${VNET_LINKS[$[$i+1]]}"
    vnetLinkExist=$(az network private-dns link vnet list -g "$RESOURCE_GROUP" \
    --zone-name $DNS_ZONE_NAME --query "[?virtualNetwork.id=='$VNET_ID'].virtualNetwork.id" -o tsv)

    if [ "$vnetLinkExist" != "$VNET_ID" ]; then
        echo "Creating Vnet Link with name: "$LINK_NAME" for the Private DNS Zone: "$DNS_ZONE_NAME" to the VNET: ""$VNET_ID"
        az network private-dns link vnet create -g "$RESOURCE_GROUP" -n $LINK_NAME \
        -z $DNS_ZONE_NAME -v "$VNET_ID" -e false
    fi
    i=$[$i+2]    
done
 ```