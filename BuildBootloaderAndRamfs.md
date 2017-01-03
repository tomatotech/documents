# Build the Bootloader and RAM Disk

### Preperations:
- [x] [Install toolchains for Amlogic platform](/develop/InstallToolchainsForAmlogicPlatform.md)
- [x] [Setup the Serial Debugging Tool](/develop/SetupSerialTool.md)


### Before starting
Create the working directories:
```sh
$ install -d ~/project/tomato/ubuntu/{linux,rootfs,archives/{ubuntu-base,debs,hwpacks},images,scripts}
$ cd project/tomato/ubuntu/
```
Clone the `utils` repository to fetch the tools for development:
```sh
$ git clone https://github.com/tomato/utils.git
```

### Build U-Boot:
Download the U-Boot source tree, take a notice of the branch `ubuntu` as the option when you run `git clone`:
```sh
$ git clone https://github.com/tomato/u-boot -b ubuntu
$ cd u-boot/
```
Build for VIM:
```sh
$ make kvim_defconfig
$ make -j8 CROSS_COMPILE=aarch64-linux-gnu-
```
*Generated files:*

* fip/u-boot.bin: bootloader blob for eMMC
* fip/u-boot.bin.sd.bin: bootloader blob for SD card
* fip/u-boot.bin.usb.tpl: bootloader blob for USB Disk(FIXME)
* fip/u-boot.bin.usb.bl2: bootloader blob for USB OTG(FIXME)

Now, you can move to [next step](/social/BuildBootloaderAndRamfs/#build-linux-kernel), or in some cases below, you might want to [create a bootable SD card](/develop/CreateBootableSDCard/) based on the newly built bootloader:

* Verify that the step above is performed correctly
* Create a bootable SD card to speed up the development process next


### Build Linux kernel
Download the Linux kernel source tree, take a notice of the branch `ubuntu` as the option when you type `git clone`:
```sh
$ git clone https://github.com/tomato/linux -b ubuntu
$ cd linux
```
Build for VIM:
```sh
$ make ARCH=arm64 kvim_defconfig
$ make -j8 ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- Image kvim.dtb modules
```
*Generated files:*

* arch/arm64/boot/dts/amlogic/kvim.dtb: device tree blob for VIM
* arch/arm64/boot/Image: Linux Image


### Build Initramfs and RAM Disk
Download the source tree and build:
```sh
$ git clone https://github.com/tomato/initrd
$ make -C initrd
```
*Generated files:*

* images/initrd.img: [initial ramfs](https://wiki.ubuntu.com/Initramfs) image

Run `mkbootimg` to generate the ramdisk image:
```sh
$ ./utils/mkbootimg --kernel linux/arch/arm64/boot/Image --ramdisk images/initrd.img -o images/boot.img
```
Well, you are almost done, and the output file `images/boot.img` is the final ramdisk you needed.

And now you can follow one of the instructions below to build the rootfs:

* [Build Ubuntu rootfs based on a prebuilt image](/social/BuildUbuntuRootfsViaPrebuiltImage.md "Easiest way")
* [Using Ubuntu-base and apt-get to build Ubuntu rootfs](/social/BuildUbuntuRootfsViaUbuntuBase.md "Recommended")
* [Compile source codes build Ubuntu rootfs](/social/BuildUbuntuRootfsViaCompiling.md "TBD")

Or if you want to have a test with the images you build aboved, please continue to read the following instruction.


### Flash the images [Optional]
Note that all the following commands prefixed as `kvim#` are executed on your target device using a [serial module](/develop/SetupSerialTool.md).

And you might need to [boot from a bootable SD](/develop/CreateBootableSDCard.md) if your target device doesn't have a bootloader installed on it yet.

Upgrade the new built bootloader into eMMC storage via TFTP [(Ensure done the right setup first)](/develop/SetupTFTPServer.md):
```
kvim# tftp 1080000 u-boot.bin
kvim# store rom_write 1080000 0 100000
```
After this, you can remove the bootable SD card from your target device, and reboot from the eMMC. 

Download the DTB blob into eMMC:
```
kvim# tftp 1080000 kvim.dtb
kvim# store dtb write 1080000
```

Load the Ramfs blob into memory and boot from the memory:
```
kvim# tftp 1080000 boot.img
kvim# bootm
```
If everything goes well, the printing should be like this when you hit enter after `boom` command:
```
kvim#bootm
ee_gate_off ...
## Booting Android Image at 0x01080000 ...
load dtb from 0x1000000 ......
   Loading Kernel Image(COMP_NONE) ... OK
   kernel loaded at 0x01080000, end = 0x01ef6be8
   Loading Ramdisk to 73a7d000, end 73ebe47e ... OK
   Loading Device Tree to 000000001fff3000, end 000000001ffff0bd ... OK

Starting kernel ...

uboot time: 58429787 us
domain-0 init dvfs: 4
[    0.000000@0] Initializing cgroup subsys cpu
[    0.000000@0] Initializing cgroup subsys cpuacct
[    0.000000@0] Linux version 3.14.29 (gouwa@Server) (gcc version 4.8.4 (Ubuntu/Linaro 4.8.4-2ubuntu1~14.04.1) ) #25 SMP PREEMPT Sun Jan 1 15:57:33 CST 2017
[    0.000000@0] CPU: AArch64 Processor [410fd034] revision 4
...
```


### Resources
* [Debian: Initramfs](https://wiki.debian.org/initramfs)
* [Linux Documentation: Initramfs & Rootfs](https://www.kernel.org/doc/Documentation/filesystems/ramfs-rootfs-initramfs.txt)
* [Reference building log of this instruction](/buildlog/BuildBootloaderAndRamfs.log)
