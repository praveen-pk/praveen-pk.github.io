---
layout: post
title:  "qemu-system-aarch64 with USB Passthrough"
date:   2021-09-10 11:30:00 +0000
categories: jekyll update
---
Below are the commmands to passthrough Host USB to aarch64 guest while using x86_64 or aarch64 Hosts.

## x86_64 Host
Following command works on ***Fedora 34*** with ***qemu-system-aarch64-5.2.0***.

On my host I have the following USB Key:

```
$ lsusb | grep SanDisk
Bus 001 Device 005: ID 0781:5406 SanDisk Corp. Cruzer Micro U3

```

Pass the USB Key to the aarch64 VM with the following command:

```
BUS_ID=1
DEV_ID=5
qemu-system-aarch64 -machine virt-5.2 -m 1024M -cpu cortex-a72 \
 -drive file=/usr/share/edk2/aarch64/QEMU_EFI-pflash.raw,if=pflash,format=raw,unit=0,readonly=on \
-usb -device usb-ehci,id=ehci \
-device usb-host,hostbus=${BUS_ID},hostaddr=${DEV_ID}   -nographic

```

## Aarch64 Host
On an aarch64 host (Rasberry Pi 4) while running ***Ubuntu 21.04*** the following command works:

```

$ lsusb  | grep SanDisk
Bus 001 Device 003: ID 0781:5406 SanDisk Corp. Cruzer Micro U3

BUS_ID=1
DEV_ID=3

$ qemu-system-aarch64 -machine virt-5.2 -m 1024M -cpu host -enable-kvm \
 -drive file=/usr/share/AAVMF/AAVMF_CODE.fd,if=pflash,format=raw,unit=0,readonly=on \
-usb -device usb-ehci,id=ehci  \
-device usb-host,hostbus=${BUS_ID},hostaddr=${DEV_ID}   -nographic

```
