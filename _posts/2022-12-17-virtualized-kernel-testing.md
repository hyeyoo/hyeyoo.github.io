---
title: Virtualized Kernel Testing using libvirt 
author: Hyeonggon Yoo
date: 2022-12-17 00:00:00 +0800
categories: [Linux Kernel]
tags: [kernel development, virtualization]
---

# Intro

This article shows how to set up your kernel test environment using libvirt, a toolkit to manage virtualization environments.

## Why use virtualized environment?

One may wonder why you use virtualization. Why not test in a local machine? Well, I mostly use cloud instances or remote servers for kernel development and testing. When you develop in a remote server, installing and testing a new kernel becomes challenging because you may break your server by mistake.

Anyway, that's enough for an introduction - let's start!

# Requirements

-   Your CPU needs to support hardware virtualization extension (i.e. Intel VT-X / AMD SVM)
-   Ubuntu is assumed for your distribution.

# Install libvirt and relevant tools

```bash
sudo apt install -y libvirt virt-manager virsh, etc)
```
Also, download pre-built ubuntu cloud image from [Ubuntu Cloud Images](https://cloud-images.ubuntu.com/) 

```bash
wget https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img
```

# Resize downloaded image
```bash
$ qemu-img info ./jammy-server-cloudimg-amd64.img
image: ./jammy-server-cloudimg-amd64.img
file format: qcow2
virtual size: 2.2 GiB (2361393152 bytes)
disk size: 636 MiB
cluster_size: 65536
Format specific information:
compat: 0.10
compression type: zlib
refcount bits: 16
```
If you check the downloaded image using 'qemu-img info', it's pretty small. Let's resize it.

## What's partitions are in this image?

Before resizing a disk partition, let's see what's root partition's name. (It's /dev/sda1)
```bash

$ virt-filesystems -h --long -a ./jammy-server-cloudimg-amd64.img
Name Type VFS Label Size Parent
/dev/sda1 filesystem ext4 cloudimg-rootfs 2.0G -
/dev/sda15 filesystem vfat UEFI 104M -
```

## Restrictions
So how to resize it? To resize, you first need to increase the disk size (virtual size) and then adjust the root partition size. VM should be shut down when resizing. 

```bash
$ qemu-img resize ./jammy-server-cloudimg-amd64.img +50G
Image resized.
image: ./jammy-server-cloudimg-amd64.img
file format: qcow2
virtual size: 52.2 GiB (56048484352 bytes)
disk size: 636 MiB
cluster_size: 65536
Format specific information:
compat: 0.10
compression type: zlib
refcount bits: 16
```

```bash

$ virt-filesystems -h --long -a ./jammy-server-cloudimg-amd64.img
Name Type VFS Label Size Parent
/dev/sda1 filesystem ext4 cloudimg-rootfs 2.0G -
/dev/sda15 filesystem vfat UEFI 104M -
```
We've just increased the disk size. But if you check the size of /dev/sda1, it remains the same. You need to adjust it to increased disk size using [virt-resize](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/virtualization_deployment_and_administration_guide/sect-guest_virtual_machine_disk_access_with_offline_tools-virt_resize_resizing_guest_virtual_machines_offline).

 
```bash
$ cp ./jammy-server-cloudimg-amd64.img ./outdisk.img # virt-resize doesn't support updating disk in-place
$ virt-resize --expand /dev/sda1 ./jammy-server-cloudimg-amd64.img ./outdisk.img
[ 0.0] Examining ./jammy-server-cloudimg-amd64.img
**********
  
Summary of changes:
/dev/sda14: This partition will be left alone.
/dev/sda15: This partition will be left alone.
/dev/sda1: This partition will be resized from 2.1G to 52.1G. The
filesystem ext4 on /dev/sda1 will be expanded using the ‘resize2fs’

method.
**********

[ 3.1] Setting up initial partition table on ./outdisk.img
[ 14.7] Copying /dev/sda14
[ 14.8] Copying /dev/sda15
[ 14.8] Copying /dev/sda1
100% ⟦▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒⟧ --:--
[ 18.4] Expanding /dev/sda1 (now /dev/sda3) using the ‘resize2fs’ method
Resize operation completed with no errors. Before deleting the old disk,
carefully check that the resized disk boots and works correctly.
```

```bash
$ virt-filesystems -h --long -a ./outdisk.img
Name Type VFS Label Size Parent
/dev/sda2 filesystem vfat UEFI 104M -
/dev/sda3 filesystem ext4 cloudimg-rootfs 50G -
```

Now the root partition is expanded. But there's one more thing left to do. Unfortunately, some Ubuntu cloud images are [incompatible](https://news.ycombinator.com/item?id=16304781) with virt-resize. It simply doesn't boot after resizing. As a workaround, [reinstall grub using virt-rescue](https://serverfault.com/questions/976792/how-to-fix-partition-table-after-virt-resize-rearranges-it-kvm) after resizing.


# Changing root password
```bash
virt-customize -a ./img --root-password 'password:root'
```

# Creating VM with virt-install


# Sharing host files with guest using virtiofs
https://libvirt.org/kbase/virtiofs.html

# Attaching a debugger to the VM