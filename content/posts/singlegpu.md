---
title: "Single GPU passthrough with AMD build"
date: 2022-06-08T21:57:09+00:00
draft: false
---
Recently I made the move to [Arch Linux](https://wiki.archlinux.org/title/Arch_Linux) as the daily driver for my gaming rig. My reason for doing so was:

-   Previously I had installed Arch on my old laptop and enjoyed the experience of having complete control over my OS
-   I’ve found that recently I’ve been spending less time playing games on my desktop and with the recent release of the Steam Deck it seemed like the right time for me to jump ship.
-   Playing with a Linux install is my idea of fun these days sadly.

However, there was one thing that held me back. I’ve been addicted to League of Legends since high school and even though there’s methods to run the game on Linux via Wine, from my experience it simply did not work.

Naturally, my next step was to undertake a much more complex endeavor to get League back onto my computer instead of trying to troubleshoot my first approach.

Enter single gpu vfio passthrough. The idea is to build a Windows 10 gaming VM via Linux’s KVM (Kernel Virtual Machine) for my League games.

To get near ‘bare metal’ performance, I needed to take some steps for my GPU to be unbound from my host OS and attach to the VM when booting it up. Then when I shut down the VM the GPU ideally rebinds back to the host.

I looked at a lot of guides over a seven day period to get a near working setup (I’ll explain later why this is “near working”). A good handful of them didn’t seem to address specific cases for my hardware, so I’m writing this to document to process for myself and help out others that may have been as confused as me.

Before we start, [This](https://gitlab.com/risingprismtv/single-gpu-passthrough/-/wikis/1) was the golden ticket for me. I started with the tutorial from SomeOrdinaryGamers, but this guide had a lot of the information that I was looking for.

## Double checking your system

You will need to make sure your hardware supports this. A couple main points that are important to check:

-   system is in UEFI mode (for my AMD based system this means CSM disabled in the bios)
-   CPU supports hardware virtualization and IOMMU
-   GPU ROM must support IOMMU, view [this link](https://www.techpowerup.com/vgabios/) and if it says your GPU supports it you’re good to go.

Here are my specs:

![](https://marcuskok.tech/wp-content/uploads/2022/06/Screenshot_20220513_232232.png)

As you can see I have a red build with an RX580 and Ryzen 5 2600X. If you have a similar build there’s a good chance what I did in this guide will work for you. However, this process is different for everyone so your mileage may vary.

_Disclaimer: for the rest of this guide the steps I describe will probably not work for Intel or Nvidia systems. I recommend looking at other guides on the internet for that._

## Enabling IOMMU

Edit the grub config found in `/etc/default/grub`. Add the options amd\_iommu=on, iommu=pt, video=efifb:off on the line GRUB\_CMDLINE\_LINUX\_DEFAULT.

```
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet iommu=pt amd_iommu=on video=efifb:off"
```

Update grub with `sudo grub-mkconfig -o /boot/grub/grub.cfg` and reboot your system.

Note: while I don’t remember the exact parameter I passed into my grub config before, I did do something that caused my boot to hang at “loading ramdisk”. To fix this, I booted from the USB I used for my install, did an `arch-chroot` into the system and undid my changes.

## Checking IOMMU groups

Use this script for viewing the groups – either copy/paste into the terminal or run as executable:

```
#!/bin/bash
shopt -s nullglob
for g in $(find /sys/kernel/iommu_groups/* -maxdepth 0 -type d | sort -V); do
    echo "IOMMU Group ${g##*/}:"
    for d in $g/devices/*; do
        echo -e "\t$(lspci -nns ${d##*/})"
    done;
done;
```

You are looking for your graphics card group. Since I am using a RX580 my group was this:

```
IOMMU Group 15:
    07:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Ellesmere [Radeon RX 470/480/570/570X/580/580X/590] [1002:67df] (rev e7)
    07:00.1 Audio device [0403]: Advanced Micro Devices, Inc. [AMD/ATI] Ellesmere HDMI Audio [Radeon RX 470/480 / 570/580/590] [1002:aaf0]
```

You should have the `VGA compatible controller` and `Audio device` together in the same group with nothing else there. Make note of this output because it is used in later steps.

## Writing hook scripts

When you launch your VM, certain scripts can be written to run on startup and shutdown.  
Having these scripts makes it so the GPU is automatically bound and unbound to whichever host it needs to go to.

There are lots of examples for start and revert scripts written by other people on the internet. Here’s what I found to work with my setup. Mileage may vary. My advice is to start with the minimum amount of lines you need and slowly build up as required. Too much in your scripts when starting will make it significantly harder to troubleshoot.

start.sh

```
#!/bin/bash
# Helpful to read output when debugging
set -x

## Load the config file with our environmental variables
source "/etc/libvirt/hooks/kvm.conf"

# Stop display manager (KDE specific)
systemctl stop sddm.service

# Unbind VTconsoles
echo 0 > /sys/class/vtconsole/vtcon0/bind
echo 0 > /sys/class/vtconsole/vtcon1/bind

# Unbind EFI-Framebuffer
echo efi-framebuffer.0 > /sys/bus/platform/drivers/efi-framebuffer/unbind

# Avoid a race condition
sleep 5

# Unload all AMD drivers
modprobe -r amdgpu

# Unbind the GPU from display driver
virsh nodedev-detach $VIRSH_GPU_VIDEO
virsh nodedev-detach $VIRSH_GPU_AUDIO

# Load VFIO kernel module
modprobe vfio
modprobe vfio_pci
modprobe vfio_iommu_type1
```

revert.sh

```
#!/bin/bash
set -x

## Load the config file
source "/etc/libvirt/hooks/kvm.conf"

# Unload VFIO-PCI Kernel Driver
modprobe -r vfio_pci
modprobe -r vfio_iommu_type1
modprobe -r vfio

# Re-Bind GPU to AMD Driver
virsh nodedev-reattach $VIRSH_GPU_VIDEO
virsh nodedev-reattach $VIRSH_GPU_AUDIO


# Rebind VT consoles
echo 1 > /sys/class/vtconsole/vtcon0/bind
echo 0 > /sys/class/vtconsole/vtcon1/bind

# Re-Bind EFI-Framebuffer
echo "efi-framebuffer.0" > /sys/bus/platform/drivers/efi-framebuffer/bind

#Loads amd drivers 
modprobe amdgpu
# Restart Display Manager
systemctl start sddm.service
```

Make sure you put these scripts in the correct paths:

```
/etc/libvirt/hooks
├── kvm.conf
├── qemu
└── qemu.d
    ├── gamingvm
    │   ├── prepare
    │   │   └── begin
    │   │       └── start.sh
    │   └── release
    │       └── end
    │           └── revert.sh
    └── win10
        ├── prepare
        │   └── begin
        │       └── start.sh
        └── release
            └── end
                └── revert.sh
```

(Note: here I have two VMs _win10_ and _gamingvm_. In this case, the scripts are the same for both)

## Setting up the VM

I’m going to keep it brief on setting up the VM. Mainly because there’s a lot of good resources online that explain it better and it’s a fairly straightforward process compared to everything else. I will however, make note of a few key details when preparing my VM.

This is assuming you are following guides regarding how to set up a virtual machine using Linux KVM.

-   I got a black screen when starting my VM and after a long time troubleshooting found that disabling rom bar in the config solved this issue.
-   I use a usb hub for my keyboard and mouse that my VM was not able to pick up on. I tried passing through the usb hub only to encounter more errors. My solution was to pass through the usb _controller_ instead
-   Like most people, I have a NVME SSD for my OS and a beefy HDD for mass storage. Initially I put the VM’s virtual disk on the HDD, but after recreating it on the SSD I saw noticeably better performance on my VM.
-   Whenever you fail always look at the VM logs and qemu logs. For example, I noticed messages in the logs complaining about not finding dmidecode – a package I had to install to get the VM working.
-   A lot of people mention patching the rom file for your graphics card online. **This was not needed in my situation**. It seems for the most part if you run an AMD card a patch is unnecessary. You will still need to define the rom file in your xml however.

## Booting it up

Hopefully with the takeaways from my experience, you’ll be able to boot with a working VM first try. However, if that is not the case just do what I did and go through the logs. Google is your best friend as doing something like this varies wildly from system to system.

My aim was to address some big questions I had as someone with a full AMD build trying to do GPU passthrough. Specifically what should go into your hook scripts (very little it turns out), and whether you need to patch the rom file for your GPU.

Doing single GPU passthrough on a VM can be very rewarding if you enjoy playing with Linux, but it takes a lot of patience and research. Between looking up errors and making changes, the whole process took me about a week. Nevertheless, at the end of the day I’m able to run Linux as my daily driver while being able to fall back on my Windows VM when I want to game with friends.
