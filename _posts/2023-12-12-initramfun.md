---
layout: post
title:  "Adventures with initramfs"
date:   2023-12-12 -1930
categories: linux
---
## Introduction
Recently, I decided to start writing posts on a blog again. I had written a few posts before I started my current job and then stopped as my workload increased and I started to do more normal things with my life. Well, I'm back and I decided to try and resurrect those old posts on a new blog. I remembered I had the files sitting on my old kali VM which was missing a lot of updates but was otherwise fine. So I powered it up, ran `apt update` and `apt upgrade` and went on with my day. An hour or so later I returned to the VM and it had froze. I was presented with the login screen but nothing could be done. I couldn't even force a restart from the kali image. Well, I had left it for an hour, surely apt was finished with updating all the tools. After all, I hadn't told it to do a full-upgrade of the system like in the kali docs. So, I made VirtualBox turn the VM off completely and I powered it back on. At was at this point I was presented with a particularly vague kernel panic message. 


![Kernel Panic at Boot, Great!](/assets/initramfs/Errors.PNG)

At this point, I knew that I could try and rescue the files from the VM and start over again with a new VM. That would have been smart, the least time consuming option and I would get a brand new VM at the end of it. This isn't what I did. I felt like I needed to fix what I broke, just so I would feel like less of an idiot. So, I ended up looking at what the issues were and if I could fix them. Researching the: `Kernel Panic- not syncing: VFS: Unable to mount root fs on unknown block` error was probably the most likely clue and sure enough, I found a bunch of posts talking about a broken initramfs setup for the kernel. 

## initramfs, something I should haven known about by now
You know how you'll maybe hear about something important but never actually look into what it is, that was me and initramfs. Apart from occasionally seeing it flash up as a word when a linux system is booting and the pretty splash screen hasn't loaded yet (if there is even a splash screen), I lived a life unburdened by the knowledge of it. Which was a shame because it's actually pretty cool. 

Okay, so, the linux kernel loads most of its drivers in via kernel modules. This is pretty useful, and it means that the kernel can support a dizzying array of hardware from the latest CPUs back to ancient network interface cards that nobody uses any more while staying comparatively simple. The use of modules means it can run on everything from the biggest supercomputers to the TV streaming stick you use for ~~hesgoals~~ streaming services. If you made a kernel that had to support the widest range of hardware without selectively loading those modules, the kernel would either be too bulky to run on a lot of lower performance systems or would be very selective with the hardware it could run on. As smart as this is though, it's not much use if the kernel doesn't have a chance to see what it needs to load in at runtime or installation. As a result, when you initialise (boot) linux a temporary filesystem is loaded into your systems RAM, allowing the kernel to see whats it working with and load in the right modules. This then allows later steps of the startup process to mount and read your main boot media, regardless of the media or interface being used. 

This **init**ialisation **RAM** **f**ile**s**ystem, or initramfs, is critical to getting the kernel running properly on the device. And, as each kernel version includes fixes and new modules, each kernel version that can run on a system needs its own initramfs to load in at boot. When this file is not present, or is invalid, the system will fail to boot. Fortunately, it is fairly simple to fix. 

## Time to Chroot 
Repairing the initramfs configuration means you need to find a way to boot into linux and mount the original boot media. Sometimes, you can tell the bootloader (for example, grub) to load in an earlier kernel version but this didn't work for me. As a result, I had to "mount" a live CD version of Debian within the VM. VirtualBox makes this easy, and after attaching the iso file on my host machine to the VM via the Virtualbox settings, I picked the CD drive option in the VirtualBox BIOS/UEFI. 

After loading up the live CD within the VM, the virtualisation layer should present the VHD as a regular internal HDD. In linux, we need to find the correct device and mount it. Using a `chroot`, we can also then map the correct directories, like `/boot/`, on the VHD to the live CD environment and run commands on the Live CD environment as if we have booted from the VHD itself. Pretty neat. Chroots are like throwaway environments that rebase the root directory for a running process. As far as that chrooted process is concerned, it cannot access anything outside of that new root directory tree. This means we can mount and chroot a shell to the VHD, and then run commands as if we're on the VHD system itself without breaking anything on the live cd. 

Based on the following [StackOverflow post](https://askubuntu.com/questions/41930/kernel-panic-not-syncing-vfs-unable-to-mount-root-fs-on-unknown-block0-0), we were able to mount and chroot into the VHD. This post also suggests running the following command: `update-initramfs -u -k <kernel name>`. The `-u` flag means we updating a pre-existing configuration for initramfs and the `-k` allows us to specify the kernel name we need to update. 

However, I haven't got the correct kernel name (we'll soon see how much of an idiot I am) so the `update-initramfs` command does not work. In the following image from the Live CD environment, the kernel I originally tried to build a initramfs for was `6.50.0-kali3-amd64`. 

![Successfully mounting the VHD](/assets/initramfs/mount%20chroot%20and%20kernel%20name.PNG)

## Who's kernel is it anyway? 
Okay, so the kernel I thought was corrupted doesn't work as the input to `update-initramfs`, but I can find the kernel version via the linux-image binaries installed on the system. `update-initramfs` also depends on files within `/boot/`.  These had the latest installed kernel (`6.5.0`) included within dpkg and within the `/boot/` directory. 

I also tried to repair any broken packages with dpkg, which further confirmed that it was 6.5.0 that was the issue. 

![DPKG begins to complain about the kernels](/assets/initramfs/dpkg%20errors.PNG)

However, this whole time I had made a big mistake. I had made a slight typo the first time I attempted to run `update-initramfs`. The first kernel name I had tried was `6.50.0-kali3-amd64`, but I meant `6.5.0-kali3-amd64`. Notice the extra 0 after the 5 when I first ran the tool. I had made a typo, and confused myself in the process. 

## Back to the Beginning
Once I had realise I had typo'd the first kernel name and inputted the correct name, the tool worked perfectly. After `update-initramfs` was actually given the name of an installed kernel, it was more than happy to read the config for the kernel and build a new initialisation filesystem. However, in order to use the new initramfs within the boot process, you would need to update the `grub` bootloader. This is easy enough, by simply running `update-grub`. 

![initramfs building a new fs](/assets/initramfs/update-initramfs%20and%20update-grub.PNG)


After restarting the VM, we see our new shiny 6.5.0 kernel running properly within Kali.

![uname lists latest kernel as running](/assets/initramfs/uname%20success.PNG)

However, running the update again still complained that the linux-image packages were broken for the 6.5.0 kernel. I tried to repair them but I came to the conclusion that removing them would be the best way forward. So, now I need to remove the latest version of the kernel that's currently running to get my package manager running again. 

![apt is still complaining about the broken packages](/assets/initramfs/apt%20upgrade%20errors.PNG) 

## Downgrade to Upgrade
As 6.5.0 is the kernel currently running on the system, it is a risky process to remove it as it is running. In fact, apt will actually throw up a ncurses style warning asking if you do want to do this. 

![DO NOT REMOVE THE KERNEL WHEN IT IS RUNNING](/assets/initramfs/kernel%20removal%20abort.PNG)

I don't want to break the VM, I just fixed after breaking it the first time, so I am gonna boot into an earlier kernel and then remove it that way. It was fairly easy to boot into the older kernel, by just choosing an older available version in grub. I wasn't able to choose an older version when the VM originally broke, so I think (but don't know) that Grub may also have broken and running the `update-grub` command fixed that too. If I could have just booted into an earlier kernel it would have saved me a lot of trouble, but unfortunately that didn't work for me. 

As it was `dpkg` that was complaining originally, I decided to use it to purge the kernel files completely. This was successful, and I was able to reboot and check that 6.5.0 was missing. I was finally able to complete an update using apt, but I chose to follow the specific kali update instructions to make sure I had the latest `sources.list` configuration.

![Finally, fixed](/assets/initramfs/succesful%20kernel%20removal.PNG)

Running `apt full-upgrade` was enough to finally get me updated to the latest version of Kali, but without the 6.5.0 kernel. Finally, the VM is fully functional and working as intended. 
