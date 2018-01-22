# Installing Arch Linux on a LUKS Encrypted Drive using LVM booting with UEFI

## Pre-flight Notes

All references to disk nodes in this document are shown as:

    /dev/sd*

You will need to change these to reflect your particular drive. To get this info use:

    #   fdisk -l

## Prepare Installation Media

[Download](https://www.archlinux.org/download/) the Arch Linux ISO and create a bootable USB drive. The simplest way to create bootable media on Linux is using the dd command:

    sudo dd bs=4M if=/path_to_arch_.iso of=/dev/sdX && sync

On Mac use Etcher or UNetBootin. On Windows use Rufus.

---

## BIOS Configuration

Hold F12 (or whatever key is used on your system) during startup to access bios. Then...

* __Make sure UEFI is ON__. Most modern systems use UEFI, so it's generally on by default.

* __Disable Secure Boot__. If secure boot is enabled it must be turned off since Linux boot loaders don't typically have digital signatures. Note that if you intend on running a dual-boot system with Windows and Linux you won't be able to use disk encryption on the partition containing Windows, as it requires secure boot.

* __Disable Fast Startup Mode__. If you are dual booting with Windows turn off Fast Startup. This feature puts Windows into hibernation when you power off. Because some systems are still active during hibernation, booting into Linux can cause various nasty problems.

---

## Boot Arch from the USB Drive

Hold F12 (or whatever key is used on your system) during startup to access startup menu. Select the USB drive and boot into Arch.

---

## Establish an Internet Connection

The most reliable way is to use a wired connection, as Arch is setup by default to connect to DHCP. However, you can usually get WiFi working by running:

    #   wifi-menu

To test your connection:

    #   ping -c 3 www.google.com

---

## Zero Hard Drive with Random Data

Optional step if you are using a hard drive with existing data. Here's how to do it using dd:

    #   dd if=/dev/urandom of=/dev/sd* status=progress

Or if you're paranoid you can use a multi-pass tool like shred.

    #   shred -vfz -n 5 /dev/sd*

---

## Partition Hard Drive

__NOTE:__ Since we're using LVM we only need two drive partitions: boot and root. The LVM will be created on root later.

First, launch __parted__ on your desired drive node;

    #   parted /dev/sd*

Then run the following commands with your particular values:

    #   (parted) mklabel gpt
    #   (parted) mkpart primary 1MiB 512MiB name 1 boot
    #   (parted) set 1 boot on
    #   (parted) mkpart primary 512MiB 100% name 2 root
    #   (parted) quit

---

## Disk Encryption

Before we setup our LVM we need to encrypt the root partition we just created.

    #   cryptsetup luksFormat -v -s 512 -h sha512 /dev/sd*

Now let's decrypt it so we can use it.

__Note__: I'm labeling this partition as "lvm". We will use this label later when we create the LVM.

    #   cryptsetup open --type luks /dev/sd* lvm

To verify our "lvm" label we can use:

    #   ls /dev/mapper/lvm

---

## LVM Setup

### Create Physical Volume

    #   pvcreate /dev/mapper/lvm

### Create Volume Group

__Note:__ I'm labelling my volume group as "vg".

    #   vgcreate vg /dev/mapper/lvm

### Create Logical Volumes

At minimum we need two volumes. One for swap, the other for root. We can additionally put home on its own volume.

__Note:__ The sizes below can be specified in megabytes (100M) or gigs (10G).

__Also__ the "L" arguments below are case sensitive. The capital L is used when you want to specify a fixed size volume, the lowercase l lets you specify percentages.

    #   lvcreate -L 4G vg -n swap
    #   lvcreate -L 20G vg -n root
    #   lvcreate -l 100%FREE vg -n home

### Create Filesystems

__Note:__ The boot partition is on the non-LVM partition.

    #   mkfs.vfat -F32 /dev/sd*
    #   mkfs.ext4 /dev/mapper/vg-root
    #   mkfs.ext4 /dev/mapper/vg-home
    #   mkswap /dev/mapper/vg-swap

### Mount the volumes

We need to create a couple directories while we're at it.

    #   mount /dev/mapper/vg-root /mnt

    #   mkdir /mnt/home
    #   mount /dev/mapper/vg-home /mnt/home

    #   mkdir /mnt/boot
    #   mount /dev/sda1 /mnt/boot

### Enable Swap

    #   swapon -s /dev/mapper/vg-swap

---

## Update mirrorlist

By default Arch has a selection of servers from various countries listed in the local mirrorlist. While you might get adequate results with the defaults, to ensure the best possible download speeds it's recommended that you update the mirrorlist with servers from your country. To do that you use an application called reflector.

### Install Reflector

Note that reflector has two dependencies (rsync and curl) which should be installed by default in Arch, but to be safe we specify them:

    #   sudo pacman -S reflector rsync curl

### Backup your local mirrorlist

    #   sudo cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.bak

### Create New Mirrorlist

Note: If you are in a different country change "United States" to your country.

    #   sudo reflector --verbose --country 'United States' -l 5 --sort rate --save /etc/pacman.d/mirrorlist

---

## Install Arch Linux

Finally we're getting somewhere!

    #   pacstrap -i /mnt base base-devel

---

### Generate fstab

We now need to update the filesystem table on the new installation. Fstab contains the association between filesystems and mountpoints.

    #   genfstab -U -p /mnt >> /mnt/etc/fstab

You can verify fstab with:

    #   cat /mnt/etc/fstab

__TO INVESTIGATE!!__ When we generated the fstab did it add our swap to it?

    /dev/mapper/vg-swap swap swap defaults 0 0

---

## Change Root

Since we're still booted via USB, in order to configure our new system we need to change root. If we don't do that, every change we make will be applied to the USB installation.

    #   arch-chroot /mnt

---

## Install and configure bootloader

While there are various bootloaders that may be used, since the Linux kernel has a built-in EFI image, all we need is a way to execute it. For that we will install systemd-boot, a minimalist boot manager:

    #   bootctl --path=/boot install

### Update the loader.conf file

Using nano we can edit the config file:

    #   nano /boot/loader/loader.conf

Make sure that __only__ the following lines are in the file:

    default arch
    timeout 3
    editor 0

__Notes:__ The timeout setting is the number of seconds the menu is displayed. The editor setting determines whether the kernel parameters are editable. For security reasons we disable this.

### Get the UUID for root

In the next step we will update the boot loader config file. But first, we need to determine the UUID of our root partition. In order to get the UUID you first need to know what device node root is on. Look it up using:

    #   fdisk -l

The device node will be something like

    #   /dev/sda2

You can now get the UUID that corresponds to the root node you just looked up using:

    #   blkid /dev/sda2

You can either write down the UUID (which could be painful given the length), or what I prefer to do is pipe the output of the above command to the config file that we will need that information in:

    #   blkid /dev/sda2 > /boot/loader/entries/arch.conf

Then open the config file in nano:

    #   nano /boot/loader/entries/arch.conf

Arrow over to the UUID and shift/arrow to highlight it. Use Ctl+K to cut the line. It will remain in the clipboard for use next.

Now delete everything in that file and add the following info. Make sure to replace __YOUR-UUID__ with the ID gathered previously (which you can paste from your clipboard using Ctrl+U).

    title   Arch Linux
    linux   /vmlinuz-linux
    initrd  /initramfs-linux.img
    options cryptdevice=UUID=YOUR-UUID:vg root=/dev/mapper/vg-root quiet rw

### Update Bootloader

    #   bootctl update

---

## Update mkinitcpio

Since we're using disk encryption we need to make sure that it gets initialized by the kernel so we can decrypt our drive prior to booting. We also need to make sure that the keyboard is available for use prior to initializing the filesystem, otherwise we will have no input device to type in our password.

Edit the following config file:

    #   nano /etc/mkintcpio.conf

Scroll down to the hooks section. It should look similar to this:

    HOOKS="base udev autodetect modconf block filesystems keyboard fsck"

Change it to this:

    HOOKS="base udev autodetect modconf block keyboard keymap encrypt lvm2 filesystems fsck"

Now update the initramfs image with our hooks change:

    #   mkinitcpio -p linux

If you're curious what modules are available as intcpio hooks:

    #   ls /usr/lib/initcpio/install

## Add NVMe to mkinitcpio

This step is only necessary if your computer is running PCIe storage rather than SATA. NVMe is a specification for accessing SSDs attached through the PCI Express bus. The Linux kernel includes an NVMe driver, so we just need to tell the kernel to load it. This is done by updating the MODULES variable in mkinitcpio (which is responsible for creating the initial ramdisk).

Edit the following config file:

    #   nano /etc/mkintcpio.conf

Add __nvme__ to the MODULES variable:

    MODULES="nvme"

Now update the initramfs image with our module change:

    #   mkinitcpio -p linux

---

## Set Language

Open the locale.gen file and uncomment your preferred language (I'm using en_US.UTF-8):

    #   nano /etc/local.gen

Now save the file and generate the locale:

    #   locale-gen

Add your language choice to the locale.conf file:

    #   echo LANG=en_US.UTF-8 > /etc/locale.conf

Export the language as an environmental shell variable:

    #   export LANG=en_US.UTF-8

---

## Set Timezone

Invoke this command to be prompted to find your timezone:

    #   tzselect

Now, use the provided TZ to create a symbolic link to /etc/localtime:

    #   ln -s /usr/share/zoneinfo/America/Denver /etc/localtime

Update the hardware clock:

    #   hwclock --systohc --utc

---

## Set Hostname

This is the name of your computer. Note: Change "arch" to whatever you want your host to be.

    #   echo arch > /etc/hostname

---

## Set Root Password

    #   passwd

---

## Create User Account

    #   useradd -m -G wheel,users -s /bin/bash <username>

And set the user password:

    #   passwd <username>

---

## Grant User Sudo Powers

Install sudo:

    #   pacman -S sudo

Then run the following command, which will open the sudoers file:

    #   EDITOR=nano visudo

Find this line and uncomment:

    #   %wheel ALL=(ALL) ALL

---

## Enable Multilib Repos and Yaourt

Multilib repos allows the user to run and build 32-bit applications on 64-bit installations. Yaourt is a package manager for AUR.

First we need to edit the pacman.conf file:

    #   nano /etc/pacman.conf

Uncomment the following lines:

    [multilib]
    Include = /etc/pacman.d/mirrorlist

Then add these lines for yaourt:

    [archlinuxfr]
    SigLevel = Never
    Server = http://repo.archlinux.fr/$arch

Save the file and exit.

### Refresh the package databases

    #   pacman -Syy

### Install Yaourt

    #   sudo pacman -S yaourt

---

## Update all packages

The installation is basically done so we now update all installed packages:

    #   pacman -Syu

---

## Reboot

You should now have a working Arch Linux installation. It doesn't have a desktop environment or any applications yet, but the base installation is done. You can now reboot and remove the USB drive:

    #   unmount -R /mnt

    #   reboot