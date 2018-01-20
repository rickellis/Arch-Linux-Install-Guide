# Arch Linux Installation Guide

#### These are the steps necessary to install <a href="https://www.archlinux.org/">Arch Linux</a> with <a href="https://en.wikipedia.org/wiki/Linux_Unified_Key_Setup">LUKS</a> disk encryption using <a href="https://en.wikipedia.org/wiki/Logical_Volume_Manager_(Linux)">Logical Volume Manager (LVM)</a> under <a href="https://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface">UEFI</a>.

---

### Before you begin, plan your disc partitioning strategy
I typically run Linux on a dual-boot system with Windows. I've also run triple boot systems, with Windows and two different Linux distros. In either case I'll generally install Windows first, then reduce the size of the partition, freeing up enough unallocated space for Linux.

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

__ALSO:__ _The amount of swap space is a function of how much ram you have. The minimum, assuming you have at least 8GB of RAM, should be 4GB, up to 1.5 times RAM. (ie, 8GB RAM = 12GB swap)._

The sizes below can be specified in megabytes (100M) or gigs (10G)

	lvcreate -n swap -L 500M arch
	lvcreate -n root -l 100%FREE  arch


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


 
### Choose download mirror (optional)
	nano /etc/pacman.d/mirrorlist
 
### Install base packages
	pacstrap -i /mnt base base-devel
 
### Configure fstab (run only once!) and verify it
	genfstab -U -p /mnt >> /mnt/etc/fstab
	nano /mnt/etc/fstab
 
### Change to your root directory
	arch-chroot /mnt
 
### Language and Time Zone settings (find and un-comment)
	nano /etc/locale.gen
	locale-gen
	echo LANG=en_US.UTF-8 > /etc/locale.conf
	export LANG=en_US.UTF-8
	 s
	ls /usr/share/zoneinfo/
	ln -s /usr/share/zoneinfo/<zone>/ /etc/localtime
	hwclock --systohc --utc
 
### Set hostname
	Note: Change "arch" to whatever you want your host to be.
	echo arch > /etc/hostname
 
### Create user
	useradd -m -G wheel,users -s /bin/bash <username>

### Set password for user
	passwd <username>

### Set root password:
	passwd
 
### Install sudo / add user to the admin-group
	pacman -S sudo bash-completion
	EDITOR=nano visudo

### and un-comment: 
	%wheel ALL=(ALL) ALL
 
 
### Enable multilib repositories: find and un-comment both lines
	nano /etc/pacman.conf
	[multilib]
	Include = /etc/pacman.d/mirrorlist

### Add Yaourt repository (same file pacman.conf)
	[archlinuxfr]
	SigLevel = Never
	Server = http://repo.archlinux.fr/$arch

### and install yaourt
	pacman -Sy yaourt customizepkg rsync
 
### Advanced Linux Sound Architecture (ALSA)
	pacman -S alsa-utils pulseaudio alsa-oss alsa-lib
	amixer sset Master unmute

### to test sound
	speaker-test -c 2
 
### Install Boot-loader (grub)

 
### Enable ethernet interface service at boot (find interface with ip link)
systemctl enable dhcpcd@.service