# Arch Linux Installation Guide

These are the steps necessary to install <a href="https://www.archlinux.org/">Arch Linux</a> with <a href="https://en.wikipedia.org/wiki/Linux_Unified_Key_Setup">LUKS</a> disk encryption using <a href="https://en.wikipedia.org/wiki/Logical_Volume_Manager_(Linux)">Logical Volume Manager (LVM)</a> under <a href="https://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface">UEFI</a>.

---

### Enable UEFI, disable Secure Boot

Hold F12 (or whatever key is used on your system) during startup to access bios. Then...

Make sure UEFI is on. Most modern systems use UEFI, so it generally is by default.

If secure boot is enabled it must be turned off since Linux boot loaders don't typically have digital signatures. Note that if you intend on running a dual-boot system with Windows and Linux you won't be able to use disk encryption on the partition containing Windows, as it requires secure boot.

---


### <a href="https://www.archlinux.org/download/">Download</a> the ISO and create a bootable USB thumb drive
The simplest way to create a bootable USB on Linux is using the dd command:

	sudo dd bs=4M if=/path_to_arch_.iso of=/dev/sdX && sync



### Boot Arch Linux from the USB thumb drive
Hold F12 during startup to access startup menu. Select the USB drive and boot into Arch.



### Establish an internet connection
The most reliable way is to use a wired connection, as Arch is setup by default to connect to DHCP. To test your wired connection:
	
	ping -c 3 www.google.com

To connect to a WiFi network:

	wifi-menu


### Delete existing disk partitions
This step is only necessary if you are using a drive with existing partitions. To see how your drive is partitioned use `fdisk -l`. If you need to remove partitions you can use parted:

	parted -s /dev/sda rm 1
	parted -s /dev/sda rm 2
	parted -s /dev/sda rm 3
	etc.

### Zero the hard drive with random data

	dd if=/dev/urandom of=/dev/sd* status=progress


### Determine which drive partition you will be using

	fdisk -l


### Partition the drive
There are a number of tools available. This is how to do it using parted:

	parted /dev/sda
	(parted) mklabel gpt
	(parted) mkpart primary 1MiB 500MiB name 1 boot
	(parted) mkpart primary 501MiB 100% name 2 root
	(parted)  quit


### Create a physical volume
	pvcreate /dev/sda2

### Create a volume group
I'm naming it "arch"

	vgcreate arch /dev/sda2

### Create logical volumes for swap and root
__NOTE:__ _The amount of swap space is a function of how much ram you have. The minimum, assuming you have at least 8GB of RAM, should be 4GB, up to 1.5 times RAM. (ie, 8GB RAM = 12GB swap)._

The sizes below can be specified in megabytes (100M) or gigs (10G)

	lvcreate -n swap -L 500M arch
	lvcreate -n root -l 100%FREE  arch


### Encrypt the root partition
	cryptsetup luksFormat -v -s 512 -h sha512 /dev/mapper/LVM-root

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