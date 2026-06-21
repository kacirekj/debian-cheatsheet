=================
Debian cheatsheet
=================


Tricks
======

-  Search inside of files::

       grep -r "hledany_text" /cesta/slozka


Package manager
===============

Full cleanup of unused data
---------------------------

-  To do cleanup::

       apt autoremove
       apt autoclean

-  To remove something including configs::

       apt purge icewm

-  To clean all Recommended and Suggested apps, which I did not want to
   have::

       apt autoremove --purge -o APT::AutoRemove::RecommendsImportant=false -o APT::AutoRemove::SuggestsImportant=false


Search
------

-  List installed packages::

       apt list --installed | grep openbox
       apt list --installed | grep -v lib | column -t

-  Search in repository::

       apt search openbox

-  List dependencies and dependants::

       apt depends openbox
       apt rdepends openbox
       aptitude why openbox

-  List apps installed by me by hand::

       cat /var/log/apt/history.log | grep -e install -e remove


X11 setup
=========

Setup default desktop
---------------------

#.  Install::

        apt install xorg
        apt install openbox
        apt install tint2
        apt install alacritty

#.  Modify ``~/.xinitrc`` for implicit behavior of the ``startx``
    command::

        #!/bin/sh

        # Allow root to start apps in session:
        xhost +SI:localuser:root >/dev/null 2>&1

        tint2 &
        exec openbox-session

    .. note::

        Openbox must be executed via session, because otherwise it will
        not load the ``/etc/xdg/`` autostart apps. This causes issue at
        least for XRDP, where ``pipewire-module-xrdp`` counts with being
        started during the X11 startup to redirect audio to XRDP.

#.  Then::

        startx


Run app as root in X-session
----------------------------

#.  When ``sudo`` is not installed, this command has to be present in
    ``.xinitrc`` or executed in X-session terminal::

        xhost +SI:localuser:root >/dev/null 2>&1

#.  Then the root app has to be executed as follows::

        su   # (without "-")
        /sbin/gparted


Stop all user-related services when user is not logged in
---------------------------------------------------------

-  It helps to have clean ``ps aux`` output::

       loginctl disable-linger debian   # For user "debian"


Setup audio
-----------

-  This should work out of box::

       apt install --no-install-recommends pipewire wireplumber pipewire-pulse pipewire-alsa


Setup XRDP
----------

-  Run::

       apt install xrdp

-  For sound to be working::

       apt install pipewire pipewire-module-xrdp

   .. note::

       Sound works out of box as far as the above script is properly
       executed from ``/etc/xdg/`` during the session startup.


Backup and recovery
===================

Backup with tar
---------------

#.  To backup currently running Debian, run this command and do not
    forget the final ``.``::

        tar --create --gzip --verbose --one-file-system --ignore-failed-read \
            --sparse --exclude=/mnt --file=/mnt/debian-backup/backup.tgz --directory=/ .


Recovery with tar
-----------------

#.  To recover, boot to Live CD.

#.  Mount backups and Debian partitions.

#.  Run::

        tar --extract --gzip --file=/mnt/debian-backup/backup.tgz --directory=/mnt/debian-rootfs/


Recover GRUB and /boot on Debian UEFI disk with LUKS root
---------------------------------------------------------

.. note::

    This can be easily made with a variant without LUKS. Because it is
    not necessary to have swap in an external partition, we do not have it
    here.

#.  Boot Live CD in UEFI mode.

#.  Mount target system::

        mount /dev/mapper/cryptroot /mnt/usb-root
        mount /dev/sdX1             /mnt/usb-root/boot/efi
        mount /dev/sdX2             /mnt/usb-root/boot

#.  Bind required system filesystems::

        mount --bind /dev           /mnt/usb-root/dev
        mount --bind /dev/pts       /mnt/usb-root/dev/pts
        mount --bind /proc          /mnt/usb-root/proc
        mount --bind /sys           /mnt/usb-root/sys

#.  Enter chroot::

        chroot /mnt/usb-root /bin/bash

#.  Fix ``/etc/crypttab``::

        cryptsetup luksUUID /dev/sdX3
        vim /etc/crypttab

    Example::

        my-name-for-mounted-luks-root  UUID=LUKS-UUID-HERE   none   luks,keyscript=decrypt_keyctl
        my-name-for-mounted-luks-swap  UUID=SWAP-UUID-HERE   none   luks,initramfs,keyscript=decrypt_keyctl

    .. note::

        Thanks to ``keyscript=decrypt_keyctl`` it will ask for password
        only once. Thanks to ``initramfs`` the swap will be unlocked during
        the password-entering phase.

    .. warning::

        ``apt install keyutils`` has to be installed, otherwise
        ``keyscript=decrypt_keyctl`` will not work.

#.  Fix ``/etc/fstab``::

        blkid
        vim /etc/fstab

    Example::

        UUID=ROOT-EXT4-UUID  /          ext4  defaults,noatime  0  1
        UUID=BOOT-UUID       /boot      ext4  defaults,noatime  0  2
        UUID=EFI-UUID        /boot/efi  vfat  umask=0077       0  1

#.  Fix new swap UUID here::

        vim /etc/initramfs-tools/conf.d/resume

    Example::

        RESUME=UUID=b6bf3907-84f5-4215-8634-e617a10e1a47

#.  Update initramfs::

        apt install cryptsetup-initramfs
        apt install keyutils
        update-initramfs -u -k all

    .. note::

        In case of swap, it may send warnings to console about old swap
        UUID not found. It should be OK.

#.  Reinstall GRUB for UEFI removable boot::

        grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=debian --removable --recheck
        update-grub

#.  Verify EFI and boot files::

        ls -R /boot/efi/EFI
        ls -l /boot

#.  Exit and unmount::

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


Add backup disk automount during startup
----------------------------------------

#.  Suppose the disk is encrypted by LUKS. Add this to
    ``/etc/crypttab``::

        crypt-debian-backup        UUID=BACKUP_DISK_LUKS_UUID none luks,initramfs,keyscript=decrypt_keyctl

#.  Add this to ``/etc/fstab``::

        UUID=BACKUP_DISK_UUID    /mnt/debian-backup     ext4    defaults,nofail     0     2

#.  Unlock the disk and update initramfs::

        cryptsetup luksOpen /dev/sdb1 crypt-debian-backup
        update-initramfs -u -k all


Creating partitions
===================

Create EFI, boot and ext4 root partition on external disk
---------------------------------------------------------

#.  Wipe old filesystem and create empty GPT table::

        wipefs -a /dev/sdX
        parted /dev/sdX --script mklabel gpt

#.  Create partitions::

        parted /dev/sdX --script mkpart my-efi   fat32    1MiB    513MiB
        parted /dev/sdX --script mkpart my-boot  ext4   513MiB   1537MiB
        parted /dev/sdX --script mkpart my-root  ext4  1537MiB  52737MiB

#.  Mark first partition as EFI::

        parted /dev/sdX --script set 1 esp on

#.  Reload partition table::

        partprobe /dev/sdX

#.  Format EFI, boot and root partitions::

        mkfs.vfat -F 32 -n  my-efi-fs   /dev/sdX1
        mkfs.ext4 -L        my-boot-fs  /dev/sdX2
        mkfs.ext4 -L        my-root-fs  /dev/sdX3

#.  Optional: To format swap, use the command as follows::

        mkswap /dev/sdXN


LUKS encryption
===============

Create fresh LUKS on partition
------------------------------

.. warning::

    This destroys existing data on the partition. Backup data first.

#.  Create LUKS container on ext4 partition::

        cryptsetup luksFormat --type luks2 /dev/sdX1
        cryptsetup open /dev/sdX1 cryptdisk
        mkfs.ext4 /dev/mapper/cryptdisk

#.  Mount it::

        mkdir -p /mnt/cryptdisk
        mount /dev/mapper/cryptdisk /mnt/cryptdisk

#.  Restore backup data into ``/mnt/cryptdisk``.


Unlock and mount existing LUKS partition
----------------------------------------

#.  Open and mount encrypted partition::

        cryptsetup luksOpen /dev/sdX1 cryptdisk
        mkdir -p /mnt/cryptdisk
        mount /dev/mapper/cryptdisk /mnt/cryptdisk

#.  Unmount and close encrypted partition::

        umount /mnt/cryptdisk
        cryptsetup luksClose cryptdisk
