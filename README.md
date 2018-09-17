# Beaglebone_GRUB_Manual

Quick installation manual how to set U-boot, GRUB, Linux and root file system based on Busybox for Beaglebone Black board.

## Motivation

[BeagleBone Black](https://beagleboard.org/black) is one of the most popular single board computer. There are many Linux distributions which are ported to this platform. In comparison to x86 systems there is no one common bootloader for ARM platforms which can uniform booting process. Each distribution for given ARM platform is equipped in it's own bootloader. In consequence the task of adjusting booting process to individual needs is tricky and it requires broad knowledge about bootloaders.

[GRUB](https://www.gnu.org/software/grub/) and [UEFI](https://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface) firmware are de facto standard for x86 platforms. GRUB can be easily adjusted to user needs. There are multiple tutorials how to use GRUB for dual booting, booting with disc decryption etc. U-Boot developers [added support for UEFI interface for U-Boot in 2017](https://www.suse.com/media/article/UEFI_on_Top_of_U-Boot.pdf) and they enabled porting GRUB to large array of ARM boards.

The main goal of the following article is to show how to configure GRUB with Linux on BeagleBone platform. You can find below step-by-step instruction how to set up U-Boot, GRUB and Linux from scratches.
