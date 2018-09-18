# GRUB for BeagleBone Black

Quick installation manual how to set U-boot, GRUB, Linux and root file system based on Busybox for Beaglebone Black board.

## Motivation

[BeagleBone Black](https://beagleboard.org/black) is one of the most popular single board computer. There are many Linux distributions which are ported to this platform. In comparison to x86 systems there is no one common bootloader for ARM platforms which can uniform booting process. Each distribution for given ARM platform is equipped in it's own bootloader. In consequence the task of adjusting booting process to individual needs is tricky and it requires broad knowledge about bootloaders.

[GRUB](https://www.gnu.org/software/grub/) and [UEFI](https://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface) firmware are de facto standard for x86 platforms. GRUB can be easily adjusted to user needs. There are multiple tutorials how to use GRUB for dual booting, booting with disc decryption etc. U-Boot developers [added support for UEFI interface for U-Boot in 2017](https://www.suse.com/media/article/UEFI_on_Top_of_U-Boot.pdf) and they enabled porting GRUB to large array of ARM boards.

The main goal of the following article is to show how to configure GRUB with Linux on BeagleBone platform. You can find below step-by-step instruction how to set up U-Boot, GRUB and Linux from scratches.

## Prerequisites

The following commands were tested on Ubuntu 18.04. Some of the commands require root account priviliges. Besides that several packages need to be installed:

```bash
sudo apt install flex bison gcc-arm-linux-gnueabi mtools parted mtd-utils e2fsprogs pigz
```

## Booting process

In comparison to x86 platforms BeagleBone Black platform has no firmware which can provide UEFI functionality which is required by GRUB. As it was stated before, U-Boot can provide UEFI functionality. In consequence, U-Boot shoudl be loaded firstly and then GRUB will load Linux system.

