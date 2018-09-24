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


## Building binaries

All binaries are build on Ubuntu 18.04 x86_64 platform. Building process does not contain generation of configuration files. It will be done in the next steps. Before building procedures starts it is worth to create a workspace directory where all binaries will be put:

```bash
mkdir workspace
cd workspace
```

### Building U-Boot

U-Boot is the first order bootloader for BeagleBone. U-Boot build requires at least gcc version 6.4. If your Linux distribution does not contain valid compiler for BeagleBone you can get the newest toolchain from [Linaro website](https://www.linaro.org/latest/downloads/). The commands presented below will build patched U-Boot for BeagleBone.

```bash
export CC=arm-linux-gnueabi-
git clone https://github.com/u-boot/u-boot
cd u-boot/
git checkout v2018.09-rc2 -b tmp
wget -c https://rcn-ee.com/repos/git/u-boot-patches/v2018.09-rc2/0001-am335x_evm-uEnv.txt-bootz-n-fixes.patch
wget -c https://rcn-ee.com/repos/git/u-boot-patches/v2018.09-rc2/0002-U-Boot-BeagleBone-Cape-Manager.patch

patch -p1 < 0001-am335x_evm-uEnv.txt-bootz-n-fixes.patch
patch -p1 < 0002-U-Boot-BeagleBone-Cape-Manager.patch
make ARCH=arm CROSS_COMPILE=${CC} distclean
make ARCH=arm CROSS_COMPILE=${CC} am335x_evm_defconfig
make ARCH=arm CROSS_COMPILE=${CC}
cd ..
```
<sub>source: [Linux on ARM - BeagleBone Black section](https://www.digikey.com/eewiki/display/linuxonarm/BeagleBone+Black)</sub>

### Building GRUB

U-Boot build process generates proper bootloaders files automatically during compilation. GRUB is slightly different. Build process delivers only tools and necessary components which needs to be put together by the user in addtional step. This approach allows user to easily adjust grub image to individual needs. `grub-mkimage` tool is responsible for generating final `grub.efi` image which will be deployed to the BeagleBone SD card. Configuration of the `grub.efi` presented below assumes that:
1. GRUB configuration file (`grub.cfg`) will be located at the top of file system of the boot partition. You can specify different localization of the `grub.cfg` file by modifying the value of the `-p` parameter.
2. GRUB modules which will be available on the BeagleBone board are listed at the end of the `./grub-mkimage` command. You can skip some modules if you don't use them.
3. All available modules, which you can use, are stored in `$(pwd)/arm_build/lib/grub/arm-efi/`.

 ```bash
git clone git://git.savannah.gnu.org/grub.git
cd grub
git checkout c79ebcd18 -b tmp
mkdir -p arm_build
./autogen.sh
GRUB_TC=arm-linux-gnueabi-
./configure --host=x86_64-linux-gnu TARGET_CC=${GRUB_TC}gcc TARGET_OBJCOPY=${GRUB_TC}objcopy TARGET_STRIP=${GRUB_TC}strip TARGET_NM=${GRUB_TC}nm TARGET_RANLIB=${GRUB_TC}ranlib   --target=arm --with-platform=efi --exec-prefix=$(pwd)/arm_build/ --prefix=$(pwd)/arm_build/
make -j4
make install
./grub-mkimage  -v -p / -o grub.efi  --format=arm-efi -d $(pwd)/arm_build/lib/grub/arm-efi/  boot linux ext2 fat serial part_msdos part_gpt normal efi_gop iso9660 configfile search loadenv test cat echo gcry_sha256 halt hashsum loadenv reboot
cd ..
```

### Building Linux kernel

BeagleBone board has its own port of Linux. There is no need to change default configuration. More information about available kernels you can find on [Linux on ARM webpage](https://www.digikey.com/eewiki/display/linuxonarm/BeagleBone+Black). The following commands will build Linux kernel for BeagleBone:

```bash
git clone https://github.com/RobertCNelson/bb-kernel
cd bb-kernel
git checkout origin/am33x-v4.9 -b tmp
./build_kernel.sh
cd ..
```
<sub>source: [Linux on ARM - BeagleBone Black section](https://www.digikey.com/eewiki/display/linuxonarm/BeagleBone+Black)</sub>

### Building Busybox

[Busybox](https://busybox.net/about.html) provides basic tools for Linux root file systems. Busybox will be used to provide minimal Linux distribution. The following commands will compile Busybox and create stub of BeagleBone root file system:

```bash
git clone git://busybox.net/busybox.git
cd busybox
git checkout 130396295 -b tmp
make defconfig
LDFLAGS="--static" make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- CONFIG_PREFIX=${CC} CONFIG_PREFIX=$(pwd)/rootfs_install install
```

## Setting up system configuration files

The following commands will work under the assumption that you are still in busybox directory. `rootfs_install` subdirectory is the place where BeagleBone root file system is created. Later, it will be mapped to `ext4` system partition. There is also a need to create separate subdirectory `efi` which will be mapped into boot efi partition.

### Content of boot EFI partition

`grub.efi` image and bootloaders configuration files should be placed on boot partition. U-Boot image files (`MLO`and `u-boot.img`) will be placed at the very beginning of the disc image. They will be located before the beginning of the boot partition for the safety reason. Copying U-Boot files is done in next steps. Now we need to be sure that U-Boot and GRUB are taylored to our needs. We need to provide automatic loading of GRUB by U-Boot and correct booting sequence of Linux. Firstly we need to create temporary directory for the future boot partition:

```bash
mkdir efi
```

#### uEnv.txt

`uEnv.txt` is the file which is used to configure U-Boot. U-Boot after self-loading searches for the configuration file which specifies next booting commandsi by overwritting default content of `uenvcmd` variable. In our case U-Boot searches for uEnv.txt file in the top level of boot partition. `uEnv.txt` contains a script, which loads from SD card into RAM memory `grub.efi` image and it starts GRUB by calling `bootefi` command. `${loadaddr}` variable is predefined by U-Boot. The command below add `uEnv.txt` file to `efi` directory:

```bash
cd efi
cat <<- 'EOF' > uEnv.txt
loadgrub=load mmc 0:1 ${loadaddr} grub.efi
uenvcmd=run loadgrub; bootefi ${loadaddr}
EOF
```

#### GRUB image and GRUB configuration file

GRUB image needs to be placed on boot partition. We have decided that GRUB configuration file will be placed at the top of boot partition. You can modify it by setting different value of `-p` parameter of `grub-mkimage` command.

GRUB configuration file contains commands which will be executed by the GRUB. This file can contain different booting scenarios. You can for example set nice boot menu where you can choose which file system you want to boot. The configuration file presented below is simple and its goal is to boot Linux without any additional menu. If you wish you can freely extend this configuration file. The syntax of GRUB configuration file is platform independent. You can reuse some parts of the configuration files from other hardware platforms.

The content of the following configuration file is as follows: firstly we set root symbol to the system partition. Then we load Linux kernel with default params (we enable output UART console, we set Linux root to partition 2 and we enable read/write access). Next, we load device tree for BeagleBone. When device tree is loaded we boot the whole system.

```bash
cat <<-EOF >grub.cfg
set root=(hd0,msdos2)
linux /boot/vmlinuz root=/dev/mmcblk0p2 console=ttyS0,115200n8 rw
devicetree /boot/am335x-boneblack.dtb
boot
EOF
```

### Linux system partition

Busybox provides almost complete basic root file systems. We have decided that Busybox will be installed in `rootfs_install` directory (parameter `CONFIG_PREFIX` in `make install` command in Busybox build section). We can freely modify the Linux root file systems. Our configuration requires adding few files to make build Linux usable.

#### Mount information

Our implementation of Linux for BeagleBone consists of 2 partitions. We need to add information where they should be placed. This information is stored in `etc/fstab` file. We need to add this file to our file system:

```bash
cd ../rootfs_install
mkdir etc
cat <<- EOF > etc/fstab
/dev/mmcblk0p1  /boot/efi  auto  defaults  0  2
/dev/mmcblk0p2 / auto errors=remount-ro 0 1
EOF
```

The first partition is boot EFI partition. According to the convention it should be mounted under `/boot/efi` directory. The second partition is system partition and it should be mounted under the root directory.

#### Console devices

We need to add some console devices in order to be able to communicate with Linux. It can be done in the following way:

```bash
mkdir dev
sudo mknod dev/console c 5 1
sudo mknod dev/null c 1 3
```

#### Linux kernel and device tree file

The Busybox build does not contain information about Linux kernel. It also does not contain valid device tree file for BeagleBone. We need to add this file. These files are located in `boot` directory. According to GRUB configuration file our Linux kernel file is called `vmlinuz` and device tree file is called `am335x-boneblack.dtb`. The commands presented below add required files to `boot` directory:

```bash
mkdir -p  boot/efi
cp ../../bb-kernel/deploy/4.9.119-bone11.zImage boot/vmlinuz
cp ../../bb-kernel/KERNEL/arch/arm/boot/dts/am335x-boneblack.dtb boot/
```

#### Init script

We need to add init script which will be used when Linux is started. Our init script is simple. It mounts `proc`, `sys` and `debugfs` file systems. Then it displays welcome message and it starts Linux shell:

```bash
mkdir -p etc/init.d
mkdir -p sys
mkdir -p sys/kernel/debug
mkdir -p proc

cat <<-EOF >etc/init.d/rcS
#!/bin/sh

mount -t proc none /proc
mount -t sysfs none /sys
mount -t debugfs none /sys/kernel/debug 
cat <<!

Boot took $(cut -d' ' -f1 /proc/uptime) seconds
Welcome to Linux on BeagleBone


!
exec /bin/sh
EOF
chmod +x etc/init.d/rcS
cd ../..
```
## Setting complete disc image

We have created complete Linux file system. We need to place created file system into single `*.img` file. This image file can be easily deployed to SD card.

### Setting U-Boot
We need to set U-Boot files (`MLO` and `u-boot.img`) at the very beginning of the final image. For the safety reason we put them just before system partition. In consequence they are invisible from Linux file system and they cannot be easily modified by accidental Linux commands. It gives us guarantee that we can access U-Boot console even if Linux file system is broken. The following commands creates blank image file and they place U-Boot files at the beginning of the image:

```bash
dd if=/dev/zero of=root.img  count=524288 bs=512
dd if=./u-boot/MLO of=root.img count=1 seek=1 bs=128k conv=notrunc
dd if=./u-boot/u-boot.img of=root.img count=2 seek=1 bs=384k conv=notrunc
```

### Setting partitions on the disc image

Now, we can create partitions on the disc image. The boot partion will be formatted as `FAT` partition. EFI standard requires this type of file system. We need to put this partition with 4 MB offset. This offset prevents againt overriding U-Boot files which were deployed to disc image in the previous step. The system partition is formatted as `ext4` partition. The following commands create both partitions. They work under the assumption that /dev/loop0 is not associated with any file. In other case please choose another loop device.

```bash
sudo losetup /dev/loop0 root.img
sudo sfdisk /dev/loop0 <<-EOF
4M,64M,,*
68M,,, 
EOF

sudo partprobe /dev/loop0
sudo mkfs.fat -n BOOT /dev/loop0p1
sudo mkfs.ext4 -L rootfs -O ^metadata_csum,^64bit /dev/loop0p2
```

### Setting up content of the partitions

Partitions inside root.img file are creted. Now we need to populate them. We can do it by mounting associate loop partitions and copying the content of `busybox/efi` and `busybox/rootfs_install` into mapped loop partitions:

```bash
mkdir mnt1
mkdir mnt2
sudo mount /dev/loop0p1  mnt1/
sudo rsync -a busybox/efi/ mnt1/
sudo chown -R root:root mnt1/
sudo umount mnt1

sudo mount /dev/loop0p2  mnt2/
sudo rsync -a busybox/rootfs_install/ mnt2/
sudo chown -R root:root mnt2/
sudo umount mnt2
sudo losetup -d /dev/loop0
```

