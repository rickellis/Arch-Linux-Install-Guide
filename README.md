# Arch-Linux-Installation-Guide

This is a work in progress...


### Establish an internet connection
The most reliable way is to use a wired connection, as Arch is setup by default to connect to DHCP. To connect to a WiFi network:

	wifi-menu


### Make sure EFI is running
Most modern systems use EFI instead of MBR to boot. The disk partitioning is slightly different on EFI systems. This document assumes EFI. If EFI is running, the following command should show output:

	efivar -l

Or alternately

	ls /sys/firmware/efi/efivars

### Zero the hard drive
First, delete any existing partitions from the drive. To see how your drive is partitioned use `fdisk -l`. If you need to remove partitions use;

	parted -s /dev/sda rm 1
	parted -s /dev/sda rm 2
	parted -s /dev/sda rm 3
	etc.

Then zero the drive with random data:

	dd if=/dev/urandom of=/dev/sd* status=progress


### Determine which drive partition you will be using

	fdisk -l


### Partition the drive






### Check if your Wireless Lan Interface is working
    ip link

### Connect to your Wifi-Network and check connection to the Internet

    wpa_passphrase <passphrase> | wpa_supplicant -B -i <SSID> -c /dev/stdin
    dhcpcd 
    ping -c 3 www.google.com
 
### Format root and home partitions using
    mkfs.ext4 /dev/sdxX

### Format/Enable SWAP
    mkswap /dev/sdaZ
    swapon /dev/sdaZ

### Check partitions with this command
lsblk /dev/sdx
 
### Mount root and home partitions
mount /dev/sdxX /mnt
mkdir /mnt/home
mount /dev/sdxY /mnt/home
 
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
 
ls /usr/share/zoneinfo/
ln -s /usr/share/zoneinfo/<zone>/ /etc/localtime
hwclock --systohc --utc
 
### Set hostname
echo  > /etc/hostname
 
### Create users (create admin psswd)
### useradd -m -g [initial_group] -G [additional_groups] -s [login_shell] [username]
useradd -m -g users -G wheel,storage,power -s /bin/bash 
passwd <username>
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
pacman -S grub
grub-install --target=i386-pc --recheck /dev/sdx
pacman -S os-prober
grub-mkconfig -o /boot/grub/grub.cfg
 
### Enable ethernet interface service at boot (find interface with ip link)
systemctl enable dhcpcd@.service