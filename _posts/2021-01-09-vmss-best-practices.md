---
title: Virtal Machine Scale Sets (VMSS) best practices
date: 2021-01-14 00:00:00 +0000
description: Virtual Machine Scale Sets (VMSS) enable you to create and manage a group of load-balanced virtual machines easily. VMSS is an IaaS service usually used in the lift-and-shift scenarios or when hosting an application inside VM is the most optimal solution from a performance or Total Cost of Ownership perspective.
categories: [Virtual Machines]
tags: [VMSS]
toc: false 
header:
 teaser: "/assets/img/posts/teasers/vmss.png"
permalink: /vm/vmss-best-practices/
excerpt: Virtual Machine Scale Sets (VMSS) enable you to create and manage a group of load-balanced virtual machines easily. VMSS is an IaaS service usually used in the lift-and-shift scenarios or when hosting an application inside VM is the most optimal solution from a performance or Total Cost of Ownership perspective. Read about the top best practices you need to know when you deploy VMSS.
---
Virtual Machine Scale Sets (VMSS) enable you to create and manage a group of load-balanced virtual machines easily. VMSS is an IaaS service usually used in the lift-and-shift scenarios or when hosting an application inside VM is the most optimal solution from a performance or Total Cost of Ownership perspective.

Note: The best practices described here doesn't apply to Azure Kubernetes Service (AKS) node pools. 

- Use [zone redundant VMSS](https://docs.microsoft.com/en-us/azure/virtual-machine-scale-sets/virtual-machine-scale-sets-use-availability-zones), to automatically spread virtual machines across availability zones. With such deployment, you will get an SLA of 99.99% for your VMs. 
    
    Note: Please ensure that all the components in your stack are redundant and consider their SLA to calculate your workload's overall SLA.
- Use the [Max spreading algorithm](https://docs.microsoft.com/en-us/azure/virtual-machine-scale-sets/virtual-machine-scale-sets-use-availability-zones#availability-considerations) to spread VMs across as many fault domains as possible within each zone.
- If you scale down to one instance, the [SLA](https://azure.microsoft.com/en-gb/support/legal/sla/virtual-machines/v1_9/) for a single VM applies. If you use HDD disks, the SLA is only 95%. Upgrade to Standard SSD will give you 99.5% SLA.
- When supported, consider using [Accelerated Networking](https://docs.microsoft.com/en-us/azure/virtual-network/create-vm-accelerated-networking-cli).  You will potentially benefit from lower latency, reduced jitter, and decreased CPU utilization.
    
    Note: Always measure the network performance before and after you enable Accelerated Networking.  In some situations, you may experience worse performances. 
- Use [autoscaling](https://docs.microsoft.com/en-us/azure/virtual-machine-scale-sets/virtual-machine-scale-sets-autoscale-overview) to automatically increase or decrease the number of VM instances. Autoscaling can help you handle the load in the time window until the faulty VM instance is automatically repaired, which is a minimum of 30 minutes.
    
    Another reason why you need to configure autoscaling is that in zone-redundant deployments if an interruption occurs in one of the zones, a scale set does not automatically scale out to increase capacity. The autoscale rules would allow the scale set to respond to a loss of the VM instances in that one zone by scaling out new instances in the remaining operational zones.
    
    Create alert rules for scale-in and scale-out (failed) events.
- Enable [boot diagnostics](https://docs.microsoft.com/en-us/azure/virtual-machines/boot-diagnostics) which allows diagnosis of VM boot failures. Use a dedicated storage account per logical stack (for example, VMSS or Resource Group).
- Use the [overprovisioning](https://docs.microsoft.com/en-us/azure/virtual-machine-scale-sets/virtual-machine-scale-sets-design-overview#overprovisioning) feature for the scale set to actually spins up more VMs than you asked for. The VMSS will delete the extra VMs once the requested number of VMs are successfully provisioned. You are not billed for the extra VMs, and they do not count toward your quota limits.
- Configure [Application Health](https://docs.microsoft.com/en-us/azure/virtual-machine-scale-sets/virtual-machine-scale-sets-health-extension) monitoring which will enable also the use of automatic instance repair.
- Enable [instance termination notification](https://docs.microsoft.com/en-us/azure/virtual-machine-scale-sets/virtual-machine-scale-sets-terminate-notification). You need to check the [Azure Metadata Service - Scheduled Events](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/scheduled-events) for Terminate events and act on them (for example,  gracefully shutdown your application). 
- If you use a private Load Balancer with your VMSS, you will not have Internet connectivity. If such connectivity is needed, you need to deploy a public Load Balancer (as part of your logical stack or use a shared one) and create an outbound rule. The other two options are Azure Firewall (much more expensive but have excellent security features) and NAT gateway ( if you deploy across availability zones, not a valid option because it is not zone redundant). 
- If your instances establish a large number of TCP outbound connections to the same destination (IP_address:port combination) or many UDP outbound connections, you can potentially face the SNAT exhaustion problem. Monitor the usage of the SNAT ports at the Load Balancer level, and if needed,  add multiple external IP addresses and increase the number of allocated outbound ports for the VMSS instances.
- Use [managed identity](https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/overview) when within the VM you need to access some of the Azure resources (for example, KeyVault). Within the VM, [call the Azure Instance Metadata Service](https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/how-to-use-vm-token) to acquire an access token that you can use to access other resources. 
- Enable login to your VMs only by using a private key(s). Store the private key(s) for VM connectivity as a secret(s) in a Key Vault with enabled auditing. Make sure you are using RSA private key.
    
    Connect to your VM instances using SSH only for debugging purposes. Deploy and use [Azure Bastion](https://docs.microsoft.com/en-us/azure/bastion/bastion-overview) for that. Using an NSG at the subnet level, allow management traffic only from the Azure Bastion.
- Use the latest VM SKU. You will get better performances, usually for the same or lower price. 
- If you can commit to using your VMs over a one-year or three-year period, purchase an [Azure Reserved Virtual Machine Instances](https://azure.microsoft.com/en-us/pricing/reserved-vm-instances/). You will save around ~30% for one-year and ~50% for three-year reservations. 
- Suppose you have Software Assurance for your Windows Server licenses or have a software subscription for Red Hat Enterprise Linux or SUSE Linux Enterprise Server. In that case, you are eligible for the [Azure Hybrid Benefit](https://azure.microsoft.com/en-us/pricing/hybrid-benefit/). With this benefit, you can reuse your existing on-premises licenses in Azure, which will save you on costs


