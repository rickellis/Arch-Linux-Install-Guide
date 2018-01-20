# Arch Linux Installation Guide

#### These are the steps necessary to install <a href="https://www.archlinux.org/">Arch Linux</a> with <a href="https://en.wikipedia.org/wiki/Linux_Unified_Key_Setup">LUKS</a> disk encryption using <a href="https://en.wikipedia.org/wiki/Logical_Volume_Manager_(Linux)">Logical Volume Manager (LVM)</a> under <a href="https://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface">UEFI</a>.

__NOTE:__ This document is NOT complete! I'm in the process of consolidating notes I've made from various places. I hope to be done with this project soon.

---

### Before you begin, plan your disc partitioning strategy
I typically run Linux on a dual-boot system with Windows. I've also run triple boot systems, with Windows and two different Linux distros. In either case I'll generally install Windows first, then reduce the size of the partition, freeing up enough unallocated space for Linux. If you are installing Linux on a dedicated drive then all you will need to do is remove the existing partitions, which can be done once you've booted from the installation media. The process is described in the drive setup instructions below.

---

## BIOS Configuration

Hold F12 (or whatever key is used on your system) during startup to access bios. Then...

### Turn UEFI on

Make sure UEFI is on. Most modern systems use UEFI, so it's generally on by default.

### Disable Secure Boot

If secure boot is enabled it must be turned off since Linux boot loaders don't typically have digital signatures. Note that if you intend on running a dual-boot system with Windows and Linux you won't be able to use BitLocker disk encryption on the partition containing Windows, as it requires secure boot.

### Disable Fast Start

If you are dual booting with Windows turn off Fast Start. Fast Start puts Windows into hibernation when you power off. Because some systems are still active during hibernation, booting into Linux can cause various problems.

---


### <a href="https://www.archlinux.org/download/">Download</a> the ISO and create a bootable USB thumb drive
The simplest way to create a bootable USB on Linux is using the dd command:

	sudo dd bs=4M if=/path_to_arch_.iso of=/dev/sdX && sync

If you prefer a graphical interface, I've heard good things about <a href="https://etcher.io/">Etcher</a> and it runs on Linux, Mac, and Windows. Alternately you can use UNetbootin (on Mac or Windows) or Rufus on Windows.


### Boot Arch Linux from the USB drive
Hold F12 during startup to access startup menu. Select the USB drive and boot into Arch.


### Establish an internet connection
The most reliable way is to use a wired connection, as Arch is setup by default to connect to DHCP. However, you can usually get WiFi working by running:

	wifi-menu

To test your connection:
	
	ping -c 3 www.google.com



To view your disc partitions:

	fdisk -l


### Delete existing disk partitions
This step is only necessary if you are using a drive with existing partitions. To remove partitions you can use parted:

	parted -s /dev/sda rm 1
	parted -s /dev/sda rm 2
	parted -s /dev/sda rm 3
	etc.


### Zero the hard drive with random data

Not really necessary, but I like knowing that I'm using a drive with no data. Here's how to do it using dd:

	dd if=/dev/urandom of=/dev/sd* status=progress

Or if you're paranoid you can use a multi-pass tool like shred;

	shred -vfz -n 5 /dev/sd*

__IMPORTANT:__ _Make sure you are pointing to the partition you want to overwrite. The above are just examples_


### Partition the drive
There are a number of tools available. This is how to do it using parted.

__NOTE:__ _Since we're using LVM we only need two drive partitions. The first is a boot partition (If you're dual booting with Windows then you will already have a boot partition that can be shared with your Linux installation, so you can skip creating it), the second is the root partition where our LVM will live._

	parted /dev/sda
	(parted) mklabel gpt
	(parted) mkpart primary 1MiB 512MiB name 1 boot
	(parted) mkpart primary 512MiB 100% name 2 root
	(parted)  quit



### Create a physical volume on the root partition
	pvcreate /dev/sda2

### Create a volume group
I usually name mine "arch". If you use something else you'll need to replace it in the next three steps.

	vgcreate arch /dev/sda2

### Create logical volumes for swap and root

__NOTE:__ I typically only use two volumes (swap and root) rather than 3 (swap, root, and home) so I can encrypt the entire root volume with my home directory in it. If you prefer three volumes then you'll need to adjust all the disc operations that follow.

__ALSO:__ _The amount of swap space is a function of how much ram you have. The minimum, assuming you have at least 8GB of RAM, should be 4GB, up to 1.5 times RAM (ie, 8GB RAM = 12GB swap) if you typically run multiple RAM intensive processes simultaneously. The average user won't need much swap._

The sizes below can be specified in megabytes (100M) or gigs (10G)

	lvcreate -n swap -L 500M arch
	lvcreate -n root -l 100%FREE arch


### Encrypt the root partition 

	cryptsetup luksFormat -v -s 512 -h sha512 /dev/mapper/arch-root


### Decrypt the newly encrypted partition
	cryptsetup open /dev/mapper/arch-root root


### Create filesystems on the two partitions

	mkfs.vfat -F32 /dev/sda1
	mkfs.ext4 /dev/mapper/root


### Mount the volumes

	mount /dev/mapper/root /mnt

	mkdir /mnt/boot
	mount /dev/sda1 /mnt/boot






