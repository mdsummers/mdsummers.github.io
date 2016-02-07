---
title: "Linux on a MacBook5,1 - Lessons learned"
date:  2016-02-07 17:30:00
description: "Some takeaways from installing Antergos on an old MacBook"
keywords: macbook, macbook51, arch, linux, antergos, b43, efi, nouveau, gdm, lightdm
---

In an effort to make more of a contribution to open source, I decided to put Linux on my home rig, a late-2008 unibody MacBook. I opted for Antergos, seeing it as a quickstart into Arch with a desktop environment.

It's no secret that installing and configuring Linux on a MacBook carries a certain amount of complexity (the [Arch Wiki page](https://wiki.archlinux.org/index.php/MacBook) on the subject is huge). Here are some of the challenges I came across during my install.

## Partitioning

I picked up a 120GB SSD for £35, putting my existing SSD aside and starting afresh.

### What doesn't work

Going in without reading the Arch wiki, I opted for a separate home and root partition. The antergos installer required me to setup a /boot/efi partition, so I added one. I also added a swap partition for if I decide to suspend the system.

~~~nohighlight
partition  mountpoint  size    type  label
/dev/sda1  /boot/efi   250MiB  fat32 EFI
/dev/sda2  /           30GiB   xfs   root
/dev/sda4  /home       remain. xfs   home
/dev/sda3  -           8GiB    swap  swap
~~~

After install the system failed to boot and I was dropped into the grub shell. The following error was repeated a number of times on the screen:

~~~nohighlight
Error: not a correct xfs inode
~~~

Followed by:

~~~nohighlight
file /boot/grub/x86_64-efi/normal.mod not found
~~~

I wasn't sure whether this was a curiousity of efi, or xfs compatibility for the /boot partition ([relevant](https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=802138)).

### What works

~~~nohighlight
partition  mountpoint  size    type  label
/dev/sda1  /boot/efi   250MiB  fat32 EFI
/dev/sda2  /boot       250MiB  fat32 boot
/dev/sda3  -           adjust  swap  swap
/dev/sda4  /           30GiB   ext4  root
/dev/sda5  /home       remain. ext4  home
~~~

The Antergos installer would not persist a non-fat filesystem type for `/boot`, I would have preferred ext2.

In hindsight there should have been no need to change `/home` and `/` away from xfs.

## Graphics

~~~nohighlight
$ lspci | grep VGA
02:00.0 VGA compatible controller: NVIDIA Corporation C79 [GeForce 9400M] (rev b1)
~~~

This laptop has an Nvidia chip for graphics. I decided to use the proprietary driver to get the best performance. This was a mistake.

### What doesn't work

This age of graphics cards requires the 304xx driver series. These come in two varieties, 30xx and 304xx-lts, for the linux and linux-lts kernels.

The 304xx module with the mainline kernel simply [does not load](https://bugs.archlinux.org/task/47092) due to missing symbols no longer exported in the 4.3 series kernel.

The lts kernel loads the driver, but the issues do not stop there. Locking the screen with the default Antergos display manager (lightdm) will often show a black screen with a cursor on it. Switching to a tty not running the X Server appears to dim the backlight so much that the screen is unreadable. Recovery is not possible, it's a reboot at this point.

### What works
Sticking with the nouveau driver gives a much better experience. I haven't noticed any performance difference in watching video between the proprietary and open source drivers.

I also elected to replace lightdm with gdm.

~~~bash
sudo pacman -S gdm
sudo systemctl disable lightdm
sudo systemctl enable gdm
reboot
~~~

This appeared to reduce the time taken between login prompt and desktop display, as well as time to lock/unlock.

## Wifi

~~~nohighlight
$ lspci | grep Network
03:00.0 Network controller: Broadcom Corporation BCM4322 802.11a/b/g/n Wireless LAN Controller (rev 01)
~~~

~~~bash
yaourt -R broadcom-wl
yaourt -S b43-firmware
ifconfig -a
~~~

## Temperature sensors

This one wasn't too difficult fortunately. I installed the `lm_sensors` package and installed the freon gnome-shell extension.

## Trackpad gestures

Out of the box I didn't have any gestures configured for navigating back/forward in browsers. The options for gestures in multitouch gestures appear to be quite fragmented:

* synaptics
* [libinput](https://www.freedesktop.org/wiki/Software/libinput/)
* [xf86-input-mtrack](https://github.com/BlueDragonX/xf86-input-mtrack)

I'm currently using libinput with [libinput-gestures](https://github.com/bulletmark/libinput-gestures). This tool allows a simple configuration file where different gestures and finger counts are mapped to xdotool actions.

I still have some issues with finger detection to get to the bottom. I've never been so acutely aware of my other fingers being on the trackpad while using one to move the cursor. Right now adding a second finger to the mix will stop the cursor in its tracks. On OS X it is possible to have another static finger on the trackpad without triggering a multifinger action. There is some ground to make up.

## Remaining tasks (TODO)

* Customise the terminal (font, colors, PS1, etc)
* Power saving settings (including getting iDevices charging while the lid is closed).