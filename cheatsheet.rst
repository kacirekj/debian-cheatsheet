=================
Debian cheatsheet
=================


Tricks
======

    Search inside of files::

        grep -r "hledany_text" /cesta/slozka

    


Package manager
===============

Full cleanup of unused data
---------------------------

To do cleanup::

    apt autoremove
    apt autoclean

To remove something including configs::

    apt purge icewm

To clean all Recommended and Suggested apps (which I didn't want to have)::

    apt autoremove --purge -o APT::AutoRemove::RecommendsImportant=false -o APT::AutoRemove::SuggestsImportant=false


Search
------

List installed::

    apt list --installed | grep openbox                  
    apt list --installed | grep -v lib | column -t

Search in repository::

    apt search openbox                                   

List dependencies and dependatns::

    apt depends openbox
    apt rdepends openbow
    aptitude why openbow

To list apps installed by me by hand::

    cat /var/log/apt/history.log | grep -e install -e remove



X11 setup
=========


Setup default desktop
---------------------

1. Install::

    apt install xorg
    apt install openbox
    apt install tint2
    apt install alacrityy

2. Modify ~/.xinitrc for implicit behavior of the startx command::

    #!/bin/sh

    # Allow root to start apps in session:
    xhost +SI:localuser:root >/dev/null 2>&1

    tint2 &
    exec openbox-session

NOTE: Openbox must be executed via session, because otherwise it will 
not load the "/etc/xdg/" autostart apps. This causes issue at least for
XRDP, where pipewire-module-xrdp counts with being started during the
x11 startup to redirect audio to XRDP.

3. Then::

    startx


Run app as root i X-session
---------------------------

1. When sudo is not installed, this command has to be present in .xinitrc or executed in X session terminal::

    xhost +SI:localuser:root >/dev/null 2>&1

2. Then the root's app *have to* be executed as follow::

    su   # (without "-")
    /sbin/gparted


Stop all user-related services when user is not logged in
---------------------------------------------------------

It helps to have clean `ps aux`::

    loginctl disable-linger debian   # For user "debian"


Setup audio
-----------

Should work out of box::

    apt install --no-install-recommends pipewire wireplumber pipewire-pulse pipewire-alsa


Setup XRDP
----------

Run::

    apt instal xrdp

For sound to be working::

    apt install pipewire pipewire-module-xrdp

NOTE: Sound works out of box as far as the above script is properly executed from /etc/xdg/ during the
session startup! 



Backup and recovery
===================


Backup with tar
---------------

1. To backup currently running Debian, run this command and don't forget "."::

    tar --create --gzip --verbose --one-file-system --ignore-failed-read --sparse --exclude=/mnt --file=/mnt/debian-backup/backup.tgz --directory=/ .


Recovery with tar
-----------------

1. To recover, boot to Live CD
2. Mount backups and Debian partitions
3. Run::

    tar --extract --gzip --file=/mnt/debian-backup/backup.tgz --directory=/mnt/debian-rootfs/


Recover GRUB and /boot on Debian UEFI disk with LUKS root
---------------------------------------------------------

NOTE: This can be easily made with variant without luks
Because it's not necessary to have swal in external partition, we don't have it.

1. Boot Live CD in UEFI mode.
2. Mount target system::

    mount /dev/mapper/cryptroot /mnt/usb-root
    mount /dev/sdX1             /mnt/usb-root/boot/efi
    mount /dev/sdX2             /mnt/usb-root/boot

3. Bind required system filesystems::

    mount --bind /dev           /mnt/usb-root/dev
    mount --bind /dev/pts       /mnt/usb-root/dev/pts
    mount --bind /proc          /mnt/usb-root/proc
    mount --bind /sys           /mnt/usb-root/sys

5. Enter chroot::

    chroot /mnt/usb-root /bin/bash

6. Fix /etc/crypttab::

    cryptsetup luksUUID /dev/sdX3
    vim /etc/crypttab

    Example:

    cryptroot UUID=LUKS-UUID-HERE none luks

7. Fix /etc/fstab::

    blkid
    vim /etc/fstab

    Example:

    UUID=ROOT-EXT4-UUID  /          ext4  defaults,noatime  0  1
    UUID=BOOT-UUID       /boot      ext4  defaults,noatime  0  2
    UUID=EFI-UUID        /boot/efi  vfat  umask=0077       0  1

8. Update initramfs. NOTE: In case of SWAP, in may send warnings to console about old swap uuid not found, it shall be ok::

    apt install cryptsetup-initramfs
    update-initramfs -u -k all

9. Reinstall GRUB for UEFI removable boot::

    grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=debian --removable --recheck
    update-grub

10. Verify EFI and boot files::

    ls -R /boot/efi/EFI
    ls -l /boot

11. Exit and unmount::

    exit
    cd /
    sync

    umount /mnt/usb-root/dev/pts 2>/dev/null
    umount /mnt/usb-root/dev     2>/dev/null
    umount /mnt/usb-root/proc    2>/dev/null
    umount /mnt/usb-root/sys     2>/dev/null
    umount /mnt/usb-root/boot/efi
    umount /mnt/usb-root/boot
    umount /mnt/usb-root

    cryptsetup close cryptroot
    sync



Creating partitions
===================


Create EFI, boot and ext4 root partition on external disk
---------------------------------------------------------

1. Wipe old filesystem and create empty GPT table::

    wipefs -a /dev/sdX
    parted /dev/sdX --script mklabel gpt

2. Create partitions::

    parted /dev/sdX --script mkpart my-efi   fat32    1MiB    513MiB
    parted /dev/sdX --script mkpart my-boot  ext4   513MiB   1537MiB
    parted /dev/sdX --script mkpart my-root  ext4  1537MiB  52737MiB

3. Mark 1st partition as EFI::

    parted /dev/sdX --script set 1 esp on

8. Reload partition table::

    partprobe /dev/sdX

9. Format EFI partition::

    mkfs.vfat -F 32 -n  my-efi-fs   /dev/sdX1
    mkfs.ext4 -L        my-boot-fs  /dev/sdX2
    mkfs.ext4 -L        my-root-fs  /dev/sdX3

10. Optional: To format swap, use the command as follow::

    mkswap /dev/sdXN



LUKS encryption
===============


Create fresh LUKS on partition
------------------------------

WARNING: This destroys existing data on the partition.
Backup data first.


1. Create LUKS container pn ext4 partition::

    cryptsetup luksFormat --type luks2 /dev/sdX1
    cryptsetup open /dev/sdX1 cryptdisk
    mkfs.ext4 /dev/mapper/cryptdisk

2. Mount it::

    mkdir -p /mnt/cryptdisk
    mount /dev/mapper/cryptdisk /mnt/cryptdisk

3. Restore backup data into /mnt/cryptdisk.


Unlock and mount existing LUKS partition
----------------------------------------

1. Open and mount encrypted partition::

    cryptsetup luksOpen /dev/sdX1 cryptdisk
    mkdir -p /mnt/cryptdisk
    mount /dev/mapper/cryptdisk /mnt/cryptdisk

2. Unmount and close encrypted partition::

    umount /mnt/cryptdisk
    cryptsetup luksClose cryptdisk
