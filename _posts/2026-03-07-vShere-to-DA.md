---
layout: post
title: "From vSphere Compromise to Domain Admin: Offline NTDS Extraction"
categories: [Pentesting, RedTeam, ActiveDirectory, Vsphere]
---

## Introduction
During an internal penetration test, I obtained administrator access to the vSphere environment. While enumerating the virtual machines, I discovered that one of them hosted the Active Directory Domain Controller. With administrative privileges on the hypervisor, cloning the Domain Controller can lead to full compromise of the domain and Domain Admin access.

## Overview 
In this article, I aim to present the approach I followed during the engagement and the reasoning behind it. During an internal penetration test against an Active Directory infrastructure, it was possible to obtain administrator credentials for the vSphere platform. I first tested these credentials against domain accounts, but they were not valid; they only provided administrative access to the virtualization platform. However, as will be shown, having control over the hypervisor can still open the door to compromising the underlying Active Directory environment. 


## Initial Enumeration 
After gaining administrator access to the vSphere environment, I noticed that the Domain Controllers for the target domain were running as virtual machines. Attempting to access the Domain Controller via the “Launch Remote Console” feature was tempting, but it would have required additional authentication. 

At this initial step, I noticed that, in addition to the Virtual Machines tab, there was a Storage management section, which offered the option to download the virtual machine disk (VMDK). This immediately reminded me of a challenge from the Certified Red Team Master (CRTM) by [Altered Security](https://www.alteredsecurity.com/gcb), which involved compromising a Domain Controller by exploiting its VHDX image. Inspired by that challenge, I set the objective to download the VMDK to my local machine for further analysis.


At this point, a small obstacle appeared: since the Domain Controller was running, it was not possible to directly download the VMDK image. However, vSphere provides a feature that allows quick cloning of virtual machines even while they are running. Therefore, before downloading the disk, I had to create a clone of the Domain Controller.

![Clone](/images/DATA/vsphere_clone.png)

After the successful clone, there were two possible alternatives: download the vmdk and mount the disk on my computer, or create a virtual machine from vSphere, add the extra disk, and mount it on the created VM. 


###  First method for mounting the disk
This was the first method I tested, but I quickly realized it was too time-consuming since the VMDK file was very large.

This method consists of downloading the Domain Controller disk to a local machine and mounting it using [OSFMount](https://www.osforensics.com/tools/mount-disk-images.html).

![osfmount](/images/DATA/osf_mount.png)

### Second method for mounting the disk
The second method turned out to be much more practical. It consists of creating a new VM in vSphere. Once the VM is created, the machine must be **powered off** in order to modify its hardware configuration. After that, you can edit the VM settings and add the cloned disk by following these steps:

**Add New Device → Existing Hard Disk**

![AddDisk](/images/DATA/vsphere_addDisk.png)

After powering on the VM, the disk will not yet be mounted. To mount it, navigate to:

**Server Manager → File and Storage Services → Disks**

Once the disk appears, select it and choose the **“Bring Online”** option.

![BringOnline](/images/DATA/vsphere_2moment.png)

The disk will then be mounted, allowing access to the file system and enabling the extraction of the **SAM**, **SYSTEM**, and **NTDS.dit** files.


## Conclusion
Once these files are obtained, it is possible to extract the domain password hashes using secretsdump. From there, compromising the Domain Controller becomes relatively straightforward. In this case, obtaining Domain Admin access required little effort beyond some troubleshooting when mounting and accessing the disk.

![Secretdump](/images/DATA/secretdump.png)

This scenario also raised an interesting question regarding disk protection mechanisms in virtualized environments. Since the VMDK could be accessed and mounted offline, I became curious about the available disk encryption mechanisms for virtual machines, and how technologies such as VM or datastore encryption could help mitigate this type of attack when an attacker gains access to the hypervisor.
