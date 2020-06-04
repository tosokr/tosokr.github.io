---
title: Migrate virtual machine into availability zone
date: 2020-02-13 20:15:00 +0000
description: How to migrate an AzureVM into availability zone with powershell
categories: [Virtual Machines]
tags: [VMs,Powershell,Availability Zones]
toc: false 
header:
 teaser: "/assets/img/posts/teasers/vm.png"
permalink: /posts/migrate-virtual-machine-into-availability-zone/
---

Availability Zones is a high-availability offering that protects you from datacenter failures. Think of them as separate datacenter inside one Azure location. For example, West Europe is one location (Amsterdam, Netherlands), but inside that location, there are several datacenters. In your subscription, you can see Availability Zones labeled as 1, 2, and 3. Those numbers are just logical representations of the physical datacenters, meaning that, for example, Availability Zone 1 in West Europe for my subscription can point to different physical datacenter compared to the Availability Zone 1 in West Europe for your subscription.

When you create a VM, you have the following options for availability:
![Desktop View]({{ "/assets/img/posts/vm/VMAvailabilityOptions.png" | relative_url }})
Note that by default *No infrastructure redundancy required* is selected. If you create the VM with this option after that is not possible to move it to an Availability Set or an Availability Zone. 

When may you need this? 
Imagine, for example, you install SQL Server inside a VM, configure everything, and after that, you choose to leverage some of the high availability options for SQL server, such as database mirroring or AlwaysOn availability groups. In such a scenario, it is a best practice to store those VMs into different Availability Zone (SLA of 99.99%) or in the same Availability Set (SLA of 99.95%).

The script workflow:
1.	Stop the VM
2.	Create a snapshot of OS and data disks. From the snapshots create a managed disk in a chosen zone 
3.	Remove the VM
4.	If there is a public Basic IP address on the NIC, remove it (this is because Basic Public IP addresses are not zone aware). If this is the case for you, after executing the script manually, add a Standard Public IP address to the NIC.
5.	Create a new VM 
6.	(If you need it) Create an SQL VM object (this install the SQL IaaS agent into the VM)

Note: After you verify that everything is working, delete the old disks and the snapshots. Otherwise, you will end up paying to Microsoft for the provisioned space (if you are using SSD disks, this can be a significant sum of money).

You can download the script from my Github repository [here](https://github.com/tosokr/Azure/blob/master/VirtualMachines/changeAvailabilityZoneOfVM.ps1)
```powershell
# Set variables
$subscriptionId="" #Set to your subscription id where the VM is created
$resourceGroup = "" #Resouce group of the VM
$vmName = "" #VM Name
$location = "westeurope" #VM Location
$zone = "2" #1 2 or 3

#Login to the Azure
Login-AzAccount

#Set the subscription
Set-AzContext -Subscription $subscriptionId

# Get the details of the VM to be moved to the Availability Set
$originalVM = Get-AzVM -ResourceGroupName $resourceGroup -Name $vmName

# Stop the VM to take snapshot
Stop-AzVM -ResourceGroupName $resourceGroup -Name $vmName -Force 

# Create a SnapShot of the OS disk and then, create an Azure Disk with Zone information
$snapshotOSConfig = New-AzSnapshotConfig -SourceUri $originalVM.StorageProfile.OsDisk.ManagedDisk.Id -Location $location -CreateOption copy -SkuName Standard_ZRS
$OSSnapshot = New-AzSnapshot -Snapshot $snapshotOSConfig -SnapshotName ($originalVM.StorageProfile.OsDisk.Name + "-snapshot") -ResourceGroupName $resourceGroup 
$diskSkuOS = (Get-AzDisk -DiskName $originalVM.StorageProfile.OsDisk.Name -ResourceGroupName $originalVM.ResourceGroupName).Sku.Name

$diskConfig = New-AzDiskConfig -Location $OSSnapshot.Location -SourceResourceId $OSSnapshot.Id -CreateOption Copy -SkuName  $diskSkuOS -Zone $zone 
$OSdisk = New-AzDisk -Disk $diskConfig -ResourceGroupName $resourceGroup -DiskName ($originalVM.StorageProfile.OsDisk.Name + "zone")


# Create a Snapshot from the Data Disks and the Azure Disks with Zone information
foreach ($disk in $originalVM.StorageProfile.DataDisks) { 

   $snapshotDataConfig = New-AzSnapshotConfig -SourceUri $disk.ManagedDisk.Id -Location $location -CreateOption copy -SkuName Standard_ZRS
   $DataSnapshot = New-AzSnapshot -Snapshot $snapshotDataConfig -SnapshotName ($disk.Name + '-snapshot') -ResourceGroupName $resourceGroup

   $diskSkuData = (Get-AzDisk -DiskName $disk.Name -ResourceGroupName $originalVM.ResourceGroupName).Sku.Name
   $datadiskConfig = New-AzDiskConfig -Location $DataSnapshot.Location -SourceResourceId $DataSnapshot.Id -CreateOption Copy -SkuName $diskSkuData -Zone $zone
   $datadisk = New-AzDisk -Disk $datadiskConfig -ResourceGroupName $resourceGroup -DiskName ($disk.Name + "zone")
}

# Remove the original VM
Remove-AzVM -ResourceGroupName $resourceGroup -Name $vmName  -Force

# Create the basic configuration for the replacement VM
$newVM = New-AzVMConfig -VMName $originalVM.Name -VMSize $originalVM.HardwareProfile.VmSize -Zone $zone

# Add the pre-existed OS disk 
Set-AzVMOSDisk -VM $newVM -CreateOption Attach -ManagedDiskId $OSdisk.Id -Name $OSdisk.Name -Windows

# Add the pre-existed data disks
foreach ($disk in $originalVM.StorageProfile.DataDisks) { 
    $datadisk = Get-AzDisk -ResourceGroupName $resourceGroup -DiskName ($disk.Name + "zone")
    Add-AzVMDataDisk -VM $newVM -Name $datadisk.Name -ManagedDiskId $datadisk.Id -Caching $disk.Caching -Lun $disk.Lun -DiskSizeInGB $disk.DiskSizeGB -CreateOption Attach 
}

# Add NIC(s) and keep the same NIC as primary
# If there is a Public IP from the Basic SKU remove it because it doesn't supports zones
foreach ($nic in $originalVM.NetworkProfile.NetworkInterfaces) {  
   $netInterface = Get-AzNetworkInterface -ResourceId $nic.Id 
   $publicIPId = $netInterface.IpConfigurations[0].PublicIpAddress.Id
   $publicIP = Get-AzPublicIpAddress -Name $publicIPId.Substring($publicIPId.LastIndexOf("/")+1) 
   if ($publicIP)
   {      
      if ($publicIP.Sku.Name -eq 'Basic')
      {
         $netInterface.IpConfigurations[0].PublicIpAddress = $null
         Set-AzNetworkInterface -NetworkInterface $netInterface
      }
   }
if ($nic.Primary -eq "True")
   {
      Add-AzVMNetworkInterface -VM $newVM -Id $nic.Id -Primary
   }
   else
   {
      Add-AzVMNetworkInterface -VM $newVM -Id $nic.Id 
   }
}

# Recreate the VM
New-AzVM -ResourceGroupName $resourceGroup -Location $originalVM.Location -VM $newVM -DisableBginfoExtension

# If the machine is SQL server, create a new SQL Server object
New-AzSqlVM -ResourceGroupName $resourceGroup -Name $newVM.Name -Location $location -LicenseType PAYG 
```