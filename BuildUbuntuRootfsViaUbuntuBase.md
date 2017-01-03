# Build Ubuntu Rootfs via Ubuntu Base


### Preperations:
- [x] [Build the Bootloader and Ramfs](/social/BuildBootloaderAndRamfs.md)


### Before starting
Confirm again you have created the working directories:
```sh
$ install -d ~/project/tomato/ubuntu/{linux,rootfs,archives/{ubuntu-base,debs,hwpacks},images,scripts}
$ cd project/tomato/ubuntu/
```

Download the latest Ubuntu-base tarball from [official Ubuntu website](http://cdimage.ubuntu.com/ubuntu-base/releases):
```sh
$ wget -P archives/ubuntu-base -c http://cdimage.ubuntu.com/ubuntu-base/releases/16.04.1/release/ubuntu-base-16.04.1-base-arm64.tar.gz
```

Install the qemu so you can run chroot to arm64:
```sh
$ sudo apt-get install qemu qemu-user-static binfmt-support debootstrap
```
It would be better for you to understand the [qemu on ARM64](https://wiki.debian.org/Arm64Qemu) first.

### Build the root file system
In this instruction, I will tend to:

* Build Ubuntu-base and preliminary rootfs on host PC and burn into target eMMC storage
* Install Ubuntu-desktop and other packages on target via running `apt-get` command with a real-time approach

Replace the below value `seek=256` upto the approach you take:

* Install Ubuntu-desktop [on host PC via Chroot]("Recommend: seek=4096")
* [Build Ubuntu-base on SD card]("FIXME")

Using `dd` to create a blank image file named as `rootfs.img`:
```sh
$ dd if=/dev/zero of=images/rootfs.img bs=1M count=0 seek=256
```

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

Extract the ubuntu-base tarball into the mounted folder:
```sh
$ sudo tar -xzf archives/ubuntu-base/ubuntu-base-16.04.1-base-arm64.tar.gz -C rootfs/
```
Ubuntu-base is a minimal rootfs and formerly named as Ubuntu-core.

And now you have already completed the basic rootfs, check it:
```sh
$ ls rootfs/
bin   dev  home  lost+found  mnt  proc  run   selinux  sys  usr
boot  etc  lib   media       opt  root  sbin  srv      tmp  var
$ 
```

FIXME: add the hwpacks

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

Setup DNS resolver:
```sh
# echo "nameserver 127.0.1.1" > /etc/resolv.conf
```

Fetch the latest package lists from server:
```sh
# apt-get update
```

Upgrade [Optional]:
```sh
# apt-get upgrade
```

Install the necessary packages:
```sh
# apt-get install ifupdown net-tools
```

When everything you want to setup has been done, exit the Chroot:
```sh
# exit
```

Remember to synchronize cached writes to persistent storage:
```sh
$ sudo sync
```

### Umount
```sh
$ sudo umount rootfs/
```

### Install Ubuntu-desktop
Download the rootfs.img into eMMC storage:



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
```
A start job is running for dev-ttys0.device
[ TIME ] Timed out waiting for device dev-ttyS0.device.
[DEPEND] Dependency failed for Serial Getty on ttyS0.
```
