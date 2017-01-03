# Build Ubuntu Rootfs via a Prebuilt Image


### Preperations:
- [x] [Build the Bootloader and Ramfs](/social/BuildBootloaderAndRamfs.md)


### Before starting
Confirm again you have created the working directories:
```sh
$ install -d ~/project/tomato/ubuntu/{linux,rootfs,archives/{ubuntu-base,debs,hwpacks},images,scripts}
$ cd project/tomato/ubuntu/
```

Install the qemu so you can run chroot to arm64:
```sh
$ sudo apt-get install qemu qemu-user-static binfmt-support debootstrap
```
It would be better for you to understand the [qemu on ARM64](https://wiki.debian.org/Arm64Qemu) first.

Download the prebuilt image you wanted, here are some good-built images for your refercence:

* [Balbes150](https://forum.armbian.com/index.php/topic/2419-armbian-for-amlogic-s905-and-s905x/): Armbian-Ubuntu-16.04, with LXDE desktop environment
* [Odroid-C2](http://odroid.com/dokuwiki/doku.php?id=en:c2_release_linux_ubuntu): Ubuntu-Mate-16.04, with Gnome desktop environment

In this guidance, I will go through with Balbes150's image, and at the monent writing the installation, the latest version is `20161223`. Decompress it:
```sh
$ xz -d archives/Armbian_5.24_Amlogic-s905x_Ubuntu_xenial_3.14.29_desktop_20161223.img.xz
```

### Build the root file system
Get the size of rootfs, it's about 3.4GB in this case:
```sh
$ du -m archives/Armbian_5.24_Amlogic-s905x_Ubuntu_xenial_3.14.29_desktop_20161223.img 
3395	archives/Armbian_5.24_Amlogic-s905x_Ubuntu_xenial_3.14.29_desktop_20161223.img
$
```

Using `dd` to create a blank image file named as `rootfs.img`:
```sh
$ dd if=/dev/zero of=images/rootfs.img bs=1M count=0 seek=3584
```
*Note: the `seek` value must be larger the size of prebuilt image.*

Create linux filesystem on the newly created image:
```sh
$ sudo mkfs.ext4 -F -L ROOTFS images/rootfs.img 
```
*Note: the value of argument `-L`(here is `ROOTFS`) should match the Linux kernel's cmdline parameters: `root=LABEL=ROOTFS`.

Create a temporary folder and ensure that it is empty:
```sh
$ rm -rf rootfs && install -d rootfs
```

Loop mount the the image file created aboved:
```sh
$ sudo mount -o loop images/rootfs.img rootfs
```

Remove the unnecessary files:
```sh
$ sudo rm -rf rootfs/lost+found
```

Mount the prebuilt image:
```sh
$ xdg-open archives/Armbian_5.24_Amlogic-s905x_Ubuntu_xenial_3.14.29_desktop_20161223.img 
$ ls /media/gouwa/
BOOT  ROOTFS
$ 
```
*Tips: The image have two partitions, `BOOT` for ramdisk, and `ROOTFS` for root file system.*

And copy the whole files in prebuilt ROOTFS mounted aboved to the destination rootfs folder:
```sh
$ sudo cp -r /media/gouwa/ROOTFS/* rootfs/
```

And now you have already completed the basic rootfs, check it:
```sh
$ ls rootfs/
bin   dev  home  lost+found  mnt  proc  run   selinux  sys  usr
boot  etc  lib   media       opt  root  sbin  srv      tmp  var
$ 
```

### Chroot and setup as you want
Setup the Chroot environment:
```sh
$ sudo cp -a /usr/bin/qemu-aarch64-static rootfs/usr/bin/
```
*Tips:*

* `/usr/bin/qemu-arm-static` is for 32-bit armhf architecture 
* `/usr/bin/qemu-aarch64-static` is for 64-bit arm64 architecture

Enter the arm64 Chroot:
```sh
$ sudo chroot rootfs/
```
**Note: Now you are in the target emulator.**

The first thing must be setup a password for the root user:
```sh
# passwd root
```
Follow the prompts to complete the setup.

You might also want to create an administrator user with 'sudo' permission [Optional]:
```sh
# useradd -G sudo -m -s /bin/bash tomato
# echo tomato:tomato | chpasswd
```

Setup a hostname for your target device [Optional]:
```sh
# echo Tomato > /etc/hostname
```

Setup HDMI, add following contents in `/etc/rc.local`:
```
# cat /etc/rc.local
#!/bin/sh -e
# ... <Omit partical contents> ...
echo 720p60hz > /sys/class/display/mode
fbset -fb /dev/fb0 -g 1280 720 1280 1440 24
fbset -fb /dev/fb1 -g 32 32 32 32 32
echo 0 > /sys/class/graphics/fb0/free_scale
echo 1 > /sys/class/graphics/fb0/freescale_mode
echo 0 0 1279 719 > /sys/class/graphics/fb0/free_scale_axis
echo 0 0 1279 719 > /sys/class/graphics/fb0/window_axis
echo 0 > /sys/class/graphics/fb1/free_scale
echo 0 > /sys/class/graphics/fb0/blank

exit 0
# 
```
*Tips:*

* Balbes150's image store HDMI init script(amlogics905x_init.sh) at /BOOT partition which is a independent partition and need mounted when booting.
* In our installations, we tend to build ramdisk with `mkbootimg`, which is whole loaded into RAM by U-Boot when booting.
* My personal think that the 2nd one will bring a faster load speed.


### Umount
In the actual developement process, the modify contents may be different from the above steps.

Or your might also need to modify and try some contents many times according to different situation util everying goes fine.

Well, when everything you want to setup has been done, exit the Chroot:
```sh
# exit
```

Remember to synchronize cached writes to persistent storage:
```sh
$ sudo sync
```

Umount the temporary rootfs folder to get the new image:
```sh
$ sudo umount rootfs/
```

Remember to umount the prebuilt disks at the same time:
```sh
$ umount /media/gouwa/BOOT /media/gouwa/ROOTFS 
```

Done!

Now after some testing, you can move forward to next step to [package the image and release](/social/PackImageForEMMC.md).

### Resources
* [Debian: ARM64 Qemu](https://wiki.debian.org/Arm64Qemu)
* [Debian: ARM64Port](https://wiki.debian.org/Arm64Port)
* [Debian Installer internals](http://d-i.alioth.debian.org/doc/talks/debconf6/paper/)
* [Ubuntu base(Wiki)](https://wiki.ubuntu.com/Base)
* [Ubuntu: Serial Console Howto](https://help.ubuntu.com/community/SerialConsoleHowto)
* [Ubuntu Installer](https://wiki.ubuntu.com/Installer/Development)
* [Ubuntu Installation Guide](https://help.ubuntu.com/lts/installation-guide/arm64/index.html)
* [Top 10 Linux Desktop Environments](https://www.linux.com/news/best-linux-desktop-environments-2016)
* [Wiki: Desktop Environments](https://wiki.archlinux.org/index.php/desktop_environment)


### Known issues
* The monitor will display blank for a while after logo and desktop.
* Cannot play 4K Videos
* [Reference building log of this instruction](/buildlog/BuildUbuntuRootfsViaPrebuiltImage.log)
