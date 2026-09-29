---
date: "2026-09-29"
draft: true
title: "TrueNAS Windows 11 VM"
description: "Setting up a Windows 11 virtual machine on TrueNAS."
summary: "Running Windows 11 in a TrueNAS VM"
tags: ["TrueNAS", "Windows 11", "home lab"]
---

I'am not a Windows guy - I mostly use Linux and macOS. I even do not own a Windows machine. Therefore, there are certain days you find yourself needing to run Windows-specific software.

Below is a rundown of the steps I followed to set up a Windows 11 VM on TrueNAS.

## Prerequisites

- TrueNAS 25.10.7 - Goldeye
- Sufficient resources (CPU, memory, storage): I'am on AMD Ryzen 5 PRO 5655G (6/12) and 128GB of RAM
- A Windows 11 ISO file (downloadable from the official Microsoft website)
- KVM Guest Windows VirtIO Drivers: the `virtio-win` ISO (downloadable from the official Fedora repository)

## Links

- [Windows 11 ISO](https://www.microsoft.com/software-download/windows11)
- [VirtIO Drivers](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/latest-virtio/)
