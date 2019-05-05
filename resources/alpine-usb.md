
# backup your disk on key device

let's assume that it is on `/dev/sdb1` and you have enough free space on destination hd.

Because following command will duplicate your's device's data into `~/sdb1.img` file.
So it will take some time to back it up.
eventually you should get an output , similar to following:
```
# root@aa:/home/aa# dd if=/dev/sdb1 of=/home/aa/sdb1.img
# 31266784+0 records in
# 31266784+0 records out
# 16008593408 bytes (16 GB, 15 GiB) copied, 1646.18 s, 9.7 MB/s
```
# to restore you do use following cmd pattern:
```
# dd if=/backup/backup-mbr-sda.img of=/dev/sdb bs=446 count=1
```
# get alpine image and write it

To write a bootable USB installation medium in Linux, run the following command; /path/archlinux.iso is the path to the downloaded ISO and /dev/sdX is the path to your unmounted target USB drive:

```

cd $HOME
ASD=/dev/sdb1
dd if=$ASD of=$HOME/asd-dev-$( date +%T ).img

wget http://mirror.rackspace.com/archlinux/iso/2019.05.02/archlinux-2019.05.02-x86_64.iso
dd bs=4M if=$HOME/archlinux-2019.05.02-x86_64.iso of=$ASD status=progress && sync
```


# boot from live device

boot Up Live Installation ISO
=============================
Insert you USB or CD installation medium in the computer and boot it up. Use the computer's boot menu (usually invoked with [F12] on a PC or the [option] key on a Mac) to select the inserted boot media. When presented with the boot device selection menu, select the entry for either BIOS or UEFI as explained above.

A bootloader menu will appear with several options. Select the Boot Arch Linux (x64_86) or (i686) option as explained above. After a minimal amount of necessary drivers are loaded to RAM, you will be presented with a root Z shell prompt.
eymap/language
---------------
The default console keymap and language in the live shell are, respectively, US and US English UTF-8. For my purposes (speaking mostly English, living mostly in Wisconsin) this has always been exactly what I needed, and the steps below to change these settings have never been required.

If a different keyboard mapping is required, view the available keymaps:
+------------------------------------------------+
|# ls /usr/share/kbd/keymaps/**/*.map.gz | less  |
+------------------------------------------------+
Load the required keymap. Here, mapname is the filename of the required map without path or file extension:
+--------------------+
|# loadkeys mapname  |
+--------------------+

To change the language of the live root environment, edit /etc/locale.gen and uncomment the line containing your desired language (for US English, uncomment en_US.UTF-8 UTF-8):
+------------------------+
|# nano /etc/locale.gen  |
+------------------------+
Generate the locale information:
+--------------+
|# locale-gen  |
+--------------+

Ensure the LANG variable is set in /etc/locale.conf (localeline is the uncommented line in /etc/locale.gen):
+-------------------------------------------+
|# echo LANG=localeline > /etc/locale.conf  |
+-------------------------------------------+

Connect to the Internet
=======================
If there is an active networking cable plugged into the machine, the live shell will bring up the computer's network interface card and automatically attempt to lease an IP address from the network during bootup. To check the network connection, ping some website:
+-----------------------+
|# ping -c1 google.com  |
+-----------------------+

wired
-----
If you have an active networking cable plugged into the machine and you are unable to connect to the internet, begin troubleshooting by viewing the interface names and statuses:
+-----------+
|# ip link  |
+-----------+
Ensure the network device is powered; ethname is the name of the interface of type link/ether (most likely the name is eno1 or eno0):
+--------------------------+
|# ip link set ethname up  |
+--------------------------+
Now attempt to manually lease a DHCP IP address from the network:
+------------------+
|# dhcpcd ethname  |
+------------------+
If you are issued a lease, try to ping an outside server again:
+--------------------------+
|# ping -c1 archlinux.org  |
+--------------------------+

If you still aren't connected or were unable to lease an IP address, view all the instances of dhcpcd to see what the hell is going on:
+--------------------------------------+
|# systemctl list-units | grep dhcpcd  |
+--------------------------------------+
See detailed instance information to troubleshoot any hardware issues:
+-------------------------------------------+
|# systemctl status dhcpcd@ethname.service  |
+-------------------------------------------+
If you still can't connect to the internet, see the ArchWiki for further help.

wireless
--------
If a wired connection is unavailable or you prefer to use wifi, most wireless interfaces are supported by the drivers on the installation ISO. To check if this is possible on your current machine, see if any kernel drivers have been loaded for the wireless interface:
+--------------------------------------------+
|# lspci -k | grep -A3 'Network controller'  |
+--------------------------------------------+

If no device is present or no drivers have been loaded, you are mostly out of luck; although it technically may be possible to import the proper drivers via some other removable memory device, such a procedure is far beyond the scope of this guide. If you really want to keep pursuing this route, checkout the official Linux wireless wiki and the ArchWiki wireless page to get started.

If the drivers have been loaded, view the names of any available wireless interfaces:
+----------+
|# iw dev  |
+----------+
Now bring the wifi interface up (wifiname is the wifi interface name given by iw dev:
+---------------------------+
|# ip link set wifiname up  |
+---------------------------+
Scan for available networks:
+---------------------------------------+
|# iw dev wifiname scan | grep 'SSID:'  |
+---------------------------------------+
If you don't know what type of authentication is required by the network you want to connect to, view the full scan results:
+-------------------------------+
|# iw dev wifiname scan | less  |
+-------------------------------+

For a connection with no encryption:
+-----------------------------------------+
|# iw dev wifiname connect 'networkname'  |
+-----------------------------------------+
For a connection with WPA/WPA2 encryption (most likely scenario):
+----------------------------------------------------------------------------+
|# wpa_supplicant -i wifiname -c <(wpa_passphrase 'networkname' 'password')  |
+----------------------------------------------------------------------------+
For a connection using WEP (old, not used much anymore):
+----------------------------------------------------------+
|# iw dev wifiname connect 'networkname' key 0:'password'  |
+----------------------------------------------------------+
Once a connection is established, fork the process to the background by pressing [ctrl]+z and running bg.

Finally, attempt lease an IP address:
+-------------------+
|# dhcpcd wifiname  |
+-------------------+
Check your network connection:
+--------------------------+
|# ping -c1 archlinux.org  |
+--------------------------+
If you are unable to obtain a wireless network connection, see the ArchWiki for more information (good luck!).

system time
-----------
Once connected to the internet, turn on the network time protocol to synchronize system time:
+----------------------------+
|# timedatectl set-ntp true  |
+----------------------------+

Prepare USB Stick
=================
As mentioned earlier, this Arch Linux bootable USB stick will be compatible with both BIOS and UEFI booting modes. In order for a storage device to boot in BIOS mode, the first 512 bytes of the device's memory must contain an MBR (master boot record). For a storage device to boot in UEFI mode, a special ESP (EFI system partion) is required. Both partitions can be created using gdisk.

Before proceeding, determine the device name of the target USB drive. First, before plugging in the target USB, view the currently available block devices:
+---------+
|# lsblk  |
+---------+
Now insert the target flash drive and view the devices again. The newly detected device /dev/sdX is the name of the target USB you will use for further partitioning and formatting. Note that in the device name you will use is literally /dev/sdX where the only thing that changes is the single lowercase letter value of X. Double check that you have the correct device name /dev/sdX, lest you may repartition the internal hard drive of the machine you are on!

wipe
----
The process of zeroing out or wiping the target USB is almost always optional, and I would suggest skipping this step the first time through (it takes a while). Overwriting the USB with zeros may be needed in rare cases when previous data aligns with the newly created partition table and is interpreted as actual files (yes, this does happen!). If you have trouble installing a bootloader further on in the installation, you may have to come back to this point and wipe your USB before continuing.

Use dd to write the USB with all zeros, permanently erasing all data:
+---------------------------------------------------------------------------+
|# dd if=/dev/zero of=/dev/sdX bs=logical-sector-size seek=0 count=sectors  |
|:     status=progress                                                      |
+---------------------------------------------------------------------------+
Expect this to take a relatively long time (hour+) depending on the size of the USB.

partition
---------
There are several partitioning tools available in the live installation environment. We'll use gdisk to partition the target USB device:
+------------------+
|# gdisk /dev/sdX  |
+------------------+

First wipe the partitions on the target USB device by typing d at the interactive prompt until no partitions remain:
+--------------------------+
| Command (? for help): d  |
| No partitions            |
+--------------------------+

Create a brand new GUID partition table:
+-----------------------------------------------------------------------+
| Command (? for help): o                                               |
| This option deletes all partitions and creates a new protective MBR.  |
| Proceed? (Y/N): y                                                     |
+-----------------------------------------------------------------------+

Make a 10MB MBR partition starting in the beginning of the device's memory:
+-----------------------------------------------------------------------+
| Command (? for help): n                                               |
| Partition number (1-128, default 1):                                  |
| First sector (34-XXXXXX), default = 64) or {+-}size{KMGTP}:           |
| Last sector (64-XXXXXX), default = XXXXXX) or {+-}size{KMGTP}: +10MB  |
| Current type is 'Linux filesystem'                                    |
| Hex code or GUID (L to show codes, Enter = 8300): EF02                |
+-----------------------------------------------------------------------+

Create a 500MB ESP partition:
+------------------------------------------------------------------------+
| Command (? for help): n                                                |
| Partition number (2-128, default 2):                                   |
| First sector (34-XXXXXX), default = YYYY) or {+-}size{KMGTP}:          |
| Last sector (64-XXXXXX), default = XXXXXX) or {+-}size{KMGTP}: +500MB  |
| Current type is 'Linux filesystem'                                     |
| Hex code or GUID (L to show codes, Enter = 8300): EF00                 |
+------------------------------------------------------------------------+

Finally, allocate the remaining space to the Linux partition:
+-----------------------------------------------------------------+
| Command (? for help): n                                         |
| Partition number (3-128, default 3):                            |
| First sector (34-XXXXXX), default = YYYY) or {+-}size{KMGTP}:   |
| Last sector (64-XXXXXX), default = XXXXXX) or {+-}size{KMGTP}:  |
| Current type is 'Linux filesystem'                              |
| Hex code or GUID (L to show codes, Enter = 8300):               |
+-----------------------------------------------------------------+

Double check the new partition table:
+---------------------------+
|# Command (? for help): p  |
+---------------------------+
It should look something like to this:
+----------------------------------------------------------------------------+
| Number  Start (sector)  End (sector)  Size      Code  Name                 |
|    1              64         20543   10.0 MiB   EF02  BIOS boot partition  |
|    2           20544       1044543   500.0 MiB  EF00  EFI System           |
|    3         1044544      62521310   29.3 GiB   8300  Linux filesystem     |
+----------------------------------------------------------------------------+

If it's all good, write it to the USB stick and exit gdisk:
+--------------------------+
| Command (? for help): w  |
+--------------------------+

format
------
View the new block layout of the target USB device:
+------------------+
|# lsblk /dev/sdX  |
+------------------+
You should now have three blocks on your target USB device: a 10MB block /dev/sdX1, a 500MB block /dev/sdX2, and block taking all the remaining memory/dev/sdX3:
+---------------------------------------------+
| NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT  |
| sdX      8:112  1  7.5G  0 disk             |
| ├─sdX1   8:113  1   10M  0 part             |
| ├─sdX2   8:114  1  500M  0 part             |
| └─sdX3   8:115  1   XXG  0 part             |
+---------------------------------------------+

Do not format the /dev/sdX1 block. This is the BIOS/MBR partion.

Format the 500MB EFI system partition with a FAT32 filesystem:
+---------------------------+
|# mkfs.fat -F32 /dev/sdX2  |
+---------------------------+

Format the Linux partition with an ext4 filesystem:
+-----------------------+
|# mkfs.ext4 /dev/sdX3  |
+-----------------------+

Install Base Package Set
========================
Following the Arch Way, this guide intends to install the minimum number of packages necessary to create a portable working Linux system.

mount
-----
Mount the ext4 formatted partition as the root filesystem:
+----------------------------+
|# mkdir -p /mnt/usb         |
|# mount /dev/sdX3 /mnt/usb  |
+----------------------------+
Mount the FAT32 formatted ESP partition to boot:
+---------------------------------+
|# mkdir /mnt/usb/boot            |
|# mount /dev/sdX2 /mnt/usb/boot  |
+---------------------------------+

pacstrap
--------
Download and install the Arch Linux base packages:
+-------------------------------------+
|# pacstrap /mnt/usb base base-devel  |
+-------------------------------------+

fstab
-----
The fstab is used by Linux systems to correctly mount available disk partitions on bootup. The partitions can be identified in the fstab in several ways, and some install methods still use the standard labels (/dev/...) instead of UUIDs. This would surely be a failure point of an install on a USB stick as the standard assigned labels for removable devices are not consistent on each boot.

Toggle the -U tag to enable UUIDs as fstab source identifiers:
+----------------------------------------------+
|# genfstab -U /mnt/usb >> /mnt/usb/etc/fstab  |
+----------------------------------------------+

Check /etc/fstab, in an editor:
+---------------------------+
|# nano /mnt/usb/etc/fstab  |
+---------------------------+
You may need to manually remove entries from your host system. For additional help manually editing /etc/fstab, see the wiki.

Configure New System
====================
Your USB stick should now contain a persistent Linux system. We still need to configure a few things before it's ready to boot on its own, though.

In addition to the base packages and in the name of ultimate portability, we will also pull and install the required programs to support automatic wired networking, manual wireless networking, all common graphics cards, touchpad input devices, and laptop battery systems. We also need to make some configuration tweaks to ensure that the new system loads support for removable devices before it attempts to access the filesystems, and that it assigns consistent names to network devices regardless of the machine it is boot up on.

chroot
------
Begin by chrooting into the new system. Besides the final network settings, everything can be set within the chroot environment:
+------------------------+
|# arch-chroot /mnt/usb  |
+------------------------+

locale
------
Use tab-completion to discover your appropriate entries for your region and city:
+---------------------------------------------------------+
|# ln -sf /usr/share/zoneinfo/region/city /etc/localtime  |
+---------------------------------------------------------+
Generate /etc/adjtime:
+---------------------+
|# hwclock --systohc  |
+---------------------+

Edit /etc/locale.gen and uncomment your desired language:
+------------------------+
|# nano /etc/locale.gen  |
+------------------------+
(for US English, uncomment en_US.UTF-8 UTF-8)

Generate the locale information:
+--------------+
|# locale-gen  |
+--------------+
Set the LANG variable in /etc/locale.conf:
+-------------------------------------------+
|# echo LANG=localeline > /etc/locale.conf  |
+-------------------------------------------+
(for US English, localeline is en_US.UTF-8)

hostname
---------
Create the /etc/hostname file containing your desired valid hostname on a single line:
+---------------------------------+
|# echo hostname > /etc/hostname  |
+---------------------------------+

Open /etc/hosts in an editor:
+-------------------+
|# nano /etc/hosts  |
+-------------------+
Add the line:
+------------------------------------------------+
| 127.0.1.1    hostname.localdomain    hostname  |
+------------------------------------------------+

RAM disk image
--------------
In order to boot the Linux Kernel persistently off of a USB device, some adjustments may be necessary to the initial RAM disk image. We need to ensure that block device support is properly loaded before any attempt at loading the filesystem. This not always the way a RAM disk image is configured in a generic Linux installation, and I suspect this may be another one of the failure points in other Linux USB installations out there. To configure a custom RAM disk image, open /etc/mkinitcpio.conf in an editor:
+-----------------------------+
|# nano /etc/mkinitcpio.conf  |
+-----------------------------+
Ensure the block hook comes before the filesystems hook and directly after the udev hook like the following:
+----------------------------------------------------+
| HOOKS=(base udev block filesystems keyboard fsck)  |
+----------------------------------------------------+

Now regenerate the initial RAM disk image with the changes made:
+-----------------------+
|# mkinitcpio -p linux  |
+-----------------------+

network interface names
-----------------------
Arch Linux's basic service manager, systemd, assigns network interfaces predictable names based on the actual device hardware. This is great for just about any other type of install, but can pose some problems for the portable USB installation we're going for. To ensure that the ethernet and wifi interfaces will always be respectively named eth0 and wlan0, revert the Arch Linux USB back to traditional device naming:
+-------------------------------------------------------------+
|# ln -s /dev/null /etc/udev/rules.d/80-net-setup-link.rules  |
+-------------------------------------------------------------+

journal config
--------------
A default installation of Arch Linux is setup with systemd to continuously journal various information about current processes and write that data to storage on disk. For a persistent bootable installation on a flash memory device, however, we can change some options in journald.conf to enable journal keeping entirely in RAM (thus reducing writes to the flash device). To control where journal data is stored, open /etc/systemd/journald.conf in and editor:
+-----------------------------------+
|# nano /etc/systemd/journald.conf  |
+-----------------------------------+
To switch journal data storage to RAM, set the storage variable to volatile by ensuring the following line is uncommented:
+-------------------+
| Storage=volatile  |
+-------------------+
As an additional precaution, to ensure the operating system doesn't overfill RAM with journal data, set the max-use variable by adding the line:
+-------------------+
| SystemMaxUse=16M  |
+-------------------+

mount options
-------------
Modern filesystems are able to record various metadata (last accessed, last modified, user rights, etc.) about their files. A default filesystem mount generally keeps track of as much as this information as possible. For a persistent bootable operating system on a flash memory device, however, we should limit some of this record keeping in order to reduce writes to the flash device. Using the noatime mount option in fstab will disable the record keeping of file access times: no writes will occur when a file is read, only when it is modified.

To disable record keeping of file access times for the bootable USB, open /etc/fstab in an editor:
+-------------------+
|# nano /etc/fstab  |
+-------------------+
Change the mount options from relatime to noatime.

bootloader
----------
To enable booting the target USB stick in both boot modes, two bootloaders will need to be installed. For ease of installation, we'll install GRUB for both modes.

Install the grub and efibootmgr packages:
+-----------------------------+
|# pacman -S grub efibootmgr  |
+-----------------------------+
View the current block devices to determine the target USB device:
+---------+
|# lsblk  |
+---------+
Note the target USB device name (without any numbers) /dev/sdX.

Setup GRUB for MBR/BIOS booting mode:
+-----------------------------------------------------------------+
|# grub-install --target=i386-pc --boot-directory /boot /dev/sdX  |
+-----------------------------------------------------------------+
Setup GRUB for UEFI booting mode:
+----------------------------------------------------------+
|# grub-install --target=x86_64-efi --efi-directory /boot  |
|:     --boot-directory /boot --removable                  |
+----------------------------------------------------------+

Generate a GRUB configuration:
+----------------------------------------+
|# grub-mkconfig -o /boot/grub/grub.cfg  |
+----------------------------------------+

network support
---------------
Install the ifplugd package to configure automatic IP leasing on ethernet devices:
+---------------------+
|# pacman -S ifplugd  |
+---------------------+

Download the packages to enable wifi support with a basic command line interface:
+--------------------------------------+
|# pacman -S iw wpa_supplicant dialog  |
+--------------------------------------+

video drivers
-------------
To support most common GPUs, install all four basic open source video drivers:
+---------------------------------------------+
|# pacman -S xf86-video-ati xf86-video-intel  |
|:     xf86-video-nouveau xf86-video-vesa     |
+---------------------------------------------+

touchpad support
----------------
Install support for standard notebook touchpads:
+----------------------------------+
|# pacman -S xf86-input-synaptics  |
+----------------------------------+

battery support
---------------
Install support for checking battery charge and state:
+------------------+
|# pacman -S acpi  |
+------------------+

root password
-------------
Set the root password:
+----------+
|# passwd  |
+----------+

user account
------------
Create a new user user:
+-------------------+
|# useradd -m user  |
+-------------------+
Set user password:
+---------------+
|# passwd user  |
+---------------+

Linux isn't meant to be used with root-user privileges all the time. Enable sudo for user by creating a rule in /etc/sudoers.d/:
+------------------------------------------------------+
|# echo 'user ALL=(ALL) ALL' > /etc/sudoers.d/10-user  |
+------------------------------------------------------+

Optionally, install polkit to ease privilege escalation for various system tasks:
+--------------------+
|# pacman -S polkit  |
+--------------------+

reboot new system
-----------------
To make the final configurations to the new Arch Linux USB, reboot into the new system. Begin by exiting the chroot environment:
+--------+
|# exit  |
+--------+
Unmount the new USB drive:
+---------------------------------+
|# umount /mnt/usb/boot /mnt/usb  |
+---------------------------------+

Powerdown the computer:
+------------+
|# poweroff  |
+------------+
If you booted a installation disk or USB, remove it now.

Powerup the computer and use the machine's boot menu to boot off the newly created Arch Linux USB. Login as the root user to complete the configuration process.

ifplugd
-------
Copy the example ethernet profile to /etc/netctl/:
+-------------------------------------------------------------------+
|# cp /etc/netctl/examples/ethernet-dhcp /etc/netctl/eth0-arch_usb  |
+-------------------------------------------------------------------+

Start ifplugd to automatically connect to any available wired network:
+-----------------------------------------------+
|# systemctl start netctl-ifplugd@eth0.service  |
+-----------------------------------------------+
Enable ifplugd in future sessions:
+------------------------------------------------+
|# systemctl enable netctl-ifplugd@eth0.service  |
+------------------------------------------------+

wifi-menu
---------
To scan for available networks and attempt to make a connection, use the simple command line tool wifi-menu:
+----------------+
|# wifi-menu -o  |
+----------------+

network time protocol
--------------------
Once connected to the internet, enable network time synchronization:
+----------------------------+
|# timedatectl set-ntp true  |
+----------------------------+

logout
-------
Logout of the root user account:
+----------+
|# logout  |
+----------+

Installation Complete!
======================
Welcome to Arch Linux! First time users should probably checkout the ArchWiki's general recommendations. 

Be safe, have fun, and treat each other nice... out...
