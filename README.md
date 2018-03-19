# Installing Arch Linux on a LUKS Encrypted Drive using LVM booting with UEFI

## Pre-flight Notes

All references to disk nodes in this document are shown as:

    /dev/sd*

You will need to change these to reflect your particular drive. To get this info use:

    $   fdisk -l

## Prepare Installation Media

[Download](https://www.archlinux.org/download/) the Arch Linux ISO and create a bootable USB drive. The simplest way to create bootable media on Linux is using the dd command:

    $   sudo dd bs=4M if=/path_to_arch_.iso of=/dev/sd* && sync

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

----

## Establish an Internet Connection

The most reliable way is to use a wired connection, as Arch is setup by default to connect to DHCP. However, you can usually get WiFi working by running:

    $   wifi-menu

To test your connection:

    $   ping -c 3 www.google.com

---

## Increase Terminal Font Size

If the terminal font is too small, which can happen if you have a high res display, then install terminus fonts.

First, update pacman caches:

    $   pacman -Sy

Then install the fonts:

    $   pacman -S terminus-font

Update font cache

    $   fc-cache -fv

Set the font to a large size:

    $   setfont ter-v32b

---

## Remove existing drive partitions

If you are installing Arch on a previously used hard drive you can remove partitions using `fdisk`

First, get the drive node name containing the partition(s) you want to remove and run:

    $   fdisk /dev/sd*

Then enter "p" for the partition list:

    $   p

Then enter "d" to delete:

    $   d

You'll be prompted to enter the number corresponding to the partition you want to remove.

To commit the changes enter:

    $   w

---

## Zero Hard Drive with Random Data

Optional step if you are using a hard drive with existing data. Here's how to do it using dd:

    $   dd if=/dev/urandom of=/dev/sd* status=progress

Or if you're paranoid (and have a day or two to wait) you can use a multi-pass tool like shred.

    $   shred -vfz -n 3 /dev/sd*

---

## Partition Hard Drive

__NOTE:__ Since we're using LVM we only need two drive partitions: boot and root. The LVM will be created on root later.

First, launch __parted__ on your desired drive node:

    $   parted /dev/sd*

Then run the following commands with your particular size values:

    $   (parted) mklabel gpt
    $   (parted) mkpart primary 1MiB 512MiB name 1 boot
    $   (parted) set 1 boot on
    $   (parted) mkpart primary 512MiB 100% name 2 root
    $   (parted) quit

---

## Disk Encryption

Before we setup our LVM we need to encrypt the root partition we just created. The Arch Wiki has more information about [LUKS](https://wiki.archlinux.org/index.php/Dm-crypt/Device_encryption).

    $   cryptsetup luksFormat -v -s 512 -h sha512 /dev/sd*

Now let's decrypt it so we can use it.

__Note__: I'm labeling this partition as "lvm". We will use this label later when we create the LVM.

    $   cryptsetup open --type luks /dev/sd* lvm

To verify our "lvm" label we can use:

    $   ls /dev/mapper/lvm

---

## LVM Setup

### Create a Physical Volume

    $   pvcreate /dev/mapper/lvm

### Create a Volume Group

__Note:__ I'm labelling my volume group as "vg". If you use something else, make sure to replace every instance of it, not only in this section, but in the bootloader config section much later.

    $   vgcreate vg /dev/mapper/lvm

### Create the Logical Volumes

At minimum we need two volumes. One for swap, the other for root. We can additionally put home on its own volume.

__Note:__ The sizes below can be specified in megabytes (100M) or gigs (10G).

__Also__ the "L" arguments below are case sensitive. The capital L is used when you want to specify a fixed size volume, the lowercase l lets you specify percentages.

    $   lvcreate -L 4G vg -n swap
    $   lvcreate -L 80G vg -n root
    $   lvcreate -l 100%FREE vg -n home

### Create the Filesystems

__Note:__ The boot partition is on the non-LVM partition, so use the disk node you specified when you created that partition.

    $   mkfs.vfat -F32 /dev/sd*
    $   mkfs.ext4 /dev/mapper/vg-root
    $   mkfs.ext4 /dev/mapper/vg-home
    $   mkswap /dev/mapper/vg-swap

### Mount the volumes

We need to create a couple directories while we're at it.

    $   mount /dev/mapper/vg-root /mnt

    $   mkdir /mnt/home
    $   mount /dev/mapper/vg-home /mnt/home

    $   mkdir /mnt/boot
    $   mount /dev/sd* /mnt/boot

### Enable Swap

    $   swapon -s /dev/mapper/vg-swap

---

## Update Mirrorlist

Before we download the Arch packages we should rank the mirrorlist to ensure our download speeds are as good as possible. There are two ways to accomplish this. The first (rankmirrors) does not require any additional packages, the other (reflector) requires a package download. Both are described next:

### Rankmirrors

Open the mirrorlist file using:

    $   nano /etc/pacman.d/mirrorlist

Then make sure that only servers in your country are uncommented. Alternately, you can `shift + arrow down` to highlight lines you don't want and `Ctrl + K` to cut them. Save the file then run these two commands:

    $   sudo cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.backup
    $   rankmirrors -n 6 /etc/pacman.d/mirrorlist.backup > /etc/pacman.d/mirrorlist


### Reflector

Install __Reflector__ using:

    $   pacman -S reflector rsync curl

__NOTE:__ If you get an error regarding missing databases, synchronize them using `pacman -Sy` then try installing Reflector again.

Now generate the new mirrorlist. Note: If you are in a different country change "United States" to your country.

    $   reflector --verbose --country 'United States' -l 5 --sort rate --save /etc/pacman.d/mirrorlist

---

## Install Arch Linux

If you only want a base Arch install with no additional packages, run:

    $   pacstrap -i /mnt base base-devel

Typically I also install `git` so I can clone my post-install setup and config scripts, along with `dialog` and `wpa_supplicant` so that `wifi-menu` will work after booting into the new system. I also install `intel-ucode` to allow the Linux kernel to update the __[processor microcode](https://wiki.archlinux.org/index.php/microcode)__.

    $   pacstrap -i /mnt base base-devel git dialog wpa_supplicant intel-ucode
---

### Generate fstab

We now need to update the filesystem table on the new installation. Fstab contains the association between filesystems and mountpoints.

    $   genfstab -U -p /mnt >> /mnt/etc/fstab

You can verify fstab with:

    $   cat /mnt/etc/fstab

---

## Change Root

Since we're still booted via USB, in order to configure our new system we need to change root. If we don't do that, every change we make will be applied to the USB installation.

    $   arch-chroot /mnt

---

## Install and configure bootloader

While there are various bootloaders that may be used, since the Linux kernel has a built-in EFI image, all we need is a way to execute it. For that we will install systemd-boot:

    $   bootctl --path=/boot install

### Update the loader.conf file

Using nano we can edit the config file:

    $   nano /boot/loader/loader.conf

Make sure that __only__ the following lines are in the file:

    default arch
    timeout 3
    editor 0

__Notes:__ The timeout setting is the number of seconds the menu is displayed. The editor setting determines whether the kernel parameters are editable. For security reasons we disable this.

### Get the UUID for root

In the next step we will update the boot loader config file. But first, we need to determine the UUID of our root partition. In order to get the UUID you first need to know what device node root is on. Look it up using:

    $   fdisk -l

The device node will be something like

    $   /dev/sda2

You can now get the UUID that corresponds to the root node you just looked up using:

    $   blkid /dev/sda2

You can either write down the UUID (which is painful given the length), or what I prefer to do is pipe the output of the above command to the config file that we will need that information in:

    $   blkid /dev/sda2 > /boot/loader/entries/arch.conf

Then open the config file in nano:

    $   nano /boot/loader/entries/arch.conf

Arrow over to the UUID and shift/arrow to highlight it. Use Ctl+K to cut the line. It will remain in the clipboard for use next.

Now __delete everything__ in that file and add the following info. Make sure to replace __YOUR-UUID__ with the ID gathered previously (which you can paste from your clipboard using Ctrl+U).

    title   Arch Linux
    linux   /vmlinuz-linux
    initrd  /initramfs-linux.img
    options cryptdevice=UUID=YOUR-UUID:vg root=/dev/mapper/vg-root quiet rw

__Note:__ If you installed the `intel-ucode` package your arch.conf file will have an additional line:

    title   Arch Linux
    linux   /vmlinuz-linux
    initrd  /intel-ucode.img
    initrd  /initramfs-linux.img
    options cryptdevice=UUID=YOUR-UUID:vg root=/dev/mapper/vg-root quiet rw

### Update Bootloader

    $   bootctl update

---

## Update mkinitcpio

Since we're using disk encryption we need to make sure that the LUKS module gets initialized by the kernel so we can decrypt our drive prior to booting. We also need to make sure that the keyboard is available for use prior to initializing the filesystem, otherwise we will have no input device to type in our password.

Edit the following config file:

    $   nano /etc/mkinitcpio.conf

Scroll down to the HOOKS section. It should look similar to this:

    HOOKS=("base udev autodetect modconf block filesystems keyboard fsck")

Change it to this:

    HOOKS=("base udev autodetect modconf block keyboard keymap encrypt lvm2 filesystems fsck")

I also add consolefont to mine in order to specify a larger default font size when booted into the terminal with high res screens, as the text ends up too small.

    HOOKS=(consolefont base udev autodetect modconf block keyboard keymap encrypt lvm2 filesystems fsck)

Now update the initramfs image with our hooks change:

__NOTE:__ Before running this command, if your computer is running PCI storage, do the steps in the next section first since it involves a change in the same file we're currently in.

    $   mkinitcpio -p linux

If you're curious what modules are available as intcpio hooks:

    $   ls /usr/lib/initcpio/install

## Add NVMe to mkinitcpio

This step is only necessary if your computer is running PCIe storage rather than SATA. NVMe is a specification for accessing SSDs attached through the PCI Express bus. The Linux kernel includes an NVMe driver, so we just need to tell the kernel to load it. This is done by updating the MODULES variable in mkinitcpio which we added HOOKS to previously.

Edit the following config file:

    $   nano /etc/mkinitcpio.conf

Add __nvme__ to the MODULES variable:

    MODULES=(nvme)

Now update the initramfs image with our module change:

    $   mkinitcpio -p linux

---

## Set Language

Open the locale.gen file and uncomment your preferred language (I'm using en_US.UTF-8):

    $   nano /etc/locale.gen

Now save the file and generate the locale:

    $   locale-gen

Add your language choice to the locale.conf file:

    $   echo LANG=en_US.UTF-8 > /etc/locale.conf

Export the language as an environmental shell variable:

    $   export LANG=en_US.UTF-8

---

## Set Timezone

Run this command to find your timezone:

    $   tzselect

Now, use the timezone you just looked up to create a symbolic link to /etc/localtime. __Note:__ Be sure to change __America/Denver__ to your timezone.

    $   ln -s /usr/share/zoneinfo/America/Denver /etc/localtime

__Note:__ If you get an error that says `failed to create symbolic link '/etc/localtime': File exists`, you must first delete the localtime file:

    $   rm /etc/localtime

Update the hardware clock. I use UTC:

    $   hwclock --systohc --utc

---

## Set Hostname

This is the name of your computer. __Note:__ Change "arch" to whatever you want your host to be.

    $   echo arch > /etc/hostname

---

## Set the Root Password

    $   passwd

---

## Create a User Account

Make sure to replace &lt;username&gt; with your username.

    $   useradd -m -G wheel,users -s /bin/bash <username>

And set the user password:

    $   passwd <username>

---

## Grant User Sudo Powers

Install sudo:

    $   pacman -S sudo

Then run the following command, which will open the sudoers file:

    $   EDITOR=nano visudo

Find this line and uncomment:

    $   %wheel ALL=(ALL) ALL

---

## Enable AUR and Multilib

Open the pacman.conf file:

    $   nano /etc/pacman.conf

Uncomment:

    #[multilib]
    #Include = /etc/pacman.d/mirrorlist

Add this:

    [archlinuxfr]
    SigLevel = Never
    Server = http://repo.archlinux.fr/$arch

In that same file add (or uncomment if it's there):

    color
    ILoveCandy

## Update all packages

The installation is basically done so we now update all installed packages:

    $   pacman -Syu

---

## Reboot

__Note:__ Before rebooting, if you are using WiFi rather than a wired connection, you might want to install a __[network package](https://wiki.archlinux.org/index.php/Wireless_network_configuration)__ to allow WiFi connectivity with your new system.

You should now have a working Arch Linux installation. It doesn't have a desktop environment or any applications yet, but the base installation is done. You can now reboot and remove the USB drive:

First, exit chroot:

    $   exit

Now unmount and reboot

    $   umount -R /mnt

    $   reboot

Or, if you prefer you can shutdown. Instead of `reboot` use:

    $   poweroff


## If you are installing Arch on VirtualBox

Instead of rebooting, as indicated above, shut down the virtual machine and do the following;

Remove the ISO you booted from at:

    Settings > Storage > Controller:IDE

Make sure EFI is enabled:

    Settings > System > Enable EFI

For WiFi connectivity, first get the name of your connection using:

    $   ip link

It should be something like `enp0s3`. Then run:

    $   sudo ip link set dev enp0s3 up
    $   sudo dhcpcd enp0s3