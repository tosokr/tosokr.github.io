---
title: Migrate virtual machine into availability zone
date: 2020-02-13 20:15:00 +0000
categories: [Virtual Machines]
tags: [VMs,PowerShell,Availability Zones]
---

Availability Zones is a high-availability offering that protects you from datacenter failures. Think of them as separate datacenter inside one Azure location. For example, West Europe is one location (Amsterdam, Netherlands), but inside that location, there are several datacenters. In your subscription, you can see Availability Zones labeled as 1, 2, and 3. Those numbers are just logical representations of the physical datacenters, meaning that, for example, Availability Zone 1 in West Europe for my subscription can point to different physical datacenter compared to the Availability Zone 1 in West Europe for your subscription.

When you create a VM, you have the following options for availability:
![Desktop View]({{ "/assets/img/posts/vm/VMAvailabilityOptions.png" | relative_url }})
Note that by default No infrastructure redundancy required is selected. If you create the VM with this option after that is not possible to move it to an Availability Set or an Availability Zone. When may you need this? Imagine, for example, you install SQL Server inside a VM, configure everything and after that, you choose to leverage some of the high availability options for SQL server, such as database mirroring or AlwaysOn availability groups. In such a scenario, it is a best practice to store those VMs into different Availability Zone (SLA of 99.99%) or in the same Availability Set (SLA of 99.95%).

The script workflow:
1.	Stop the VM
2.	Create a snapshot of OS and data disks. From the snapshots create a managed disk in a chosen zone 
3.	Remove the VM
4.	If there is a public Basic IP address on the NIC, remove it (this is because Basic Public IP addresses are not zone aware). If this is the case for you, after executing the script manually, add a Standard Public IP address to the NIC.
5.	Create a new VM 
6.	(If you need it) Create an SQL VM object (this install the SQL IaaS agent into the VM)

After you verify that everything is working, delete the old disks and the snapshots. Otherwise, you will pay to Microsoft for the provisioned space (if you are using SSD disks, this can be a significant sum).

```powershell
{% remote_markdown https://raw.githubusercontent.com/tosokr/Azure/master/VirtualMachines/changeAvailabilityZoneOfVM.ps1 %}
```