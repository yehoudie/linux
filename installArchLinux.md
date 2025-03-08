# Install Arch Linux with full disk encryption (including /boot) and UEFI. #

Guide to install archlinux with full disk encryption and encrypted boot loader.


## Content
- [Desired layout](#desired-layout)
    - [Remarks](#remarks)
- [Download](#download)
- [Wiping](#wiping)
- [Preliminary steps](#preliminary-steps)
    - [Keyboard layout](#keyboard-layout)
    - [Connect to Wi-Fi](#connect-to-wi-fi)
- [Enable network time synchronization](#enable-network-time-synchronization)
- [Partitioning](#partitioning)
    - [Create partitions](#create-partitions)
    - [Initialize partitions](#initialize-partitions)
    - [Mount](#mount)
    - [Check](#check)
- [Install System](#install-system)
- [Init System](#init-system)
- [Grub](#grub)
- [Crypttab](#crypttab)
- [Done](#done)
- [Additional steps](#additional-steps)
    - [Ease of access](#ease-of-access)
    - [Intel microcode](#intel-microcode)
    - [Some additional security](#some-additional-security)
    - [Additional users](#additional-users)
    - [Internet](#internet)
    - [Dual Boot](#dual-boot)
- [Finish](#finish)
- [Use your system](#use-your-system)
- [Aux](#aux)
    - [Secure boot](#secure-boot)
    - [Temp Device](#temp-device)
- [Troubleshooting](#troubleshooting)
    - [Operation system not found](#operation-system-not-found)
    - [lvmetad.socket](#lvmetad.socket)
    - [efivars/efivarfs](#efivars/efivarfs)
    - [Not Bootable](#not-bootable)
        - [Grub and Dell](#grub-and-dell)
        - [Missing Modules](#missing-modules)
    - [Grub and German Keyboard](#grub-and-german-keyboard)
    - [Grub cmd](#grub-cmd)
    - [No Hibernation Device found](#no-hibernation-device-found)
    - [Battery](#battery)
    - [Stuff](#stuff)
- [Recovery](#recovery)
- [Keys](#keys)
- [LVM](#lvm)
    - [Volume Group Info](#volume-group-info)
    - [Logical Volume Info](#logical-volume-info)
    - [Resize Logical Volume](#resize-logical-volume)
- [Kernels](#kernels)
- [Links](#links)
    - [Installation](#installation)
    - [Crypto](#crypto)

## Desired layout
```text
+---------------+----------------+----------------+----------------+
|ESP partition  |Boot partition  |Volume 1        |Volume 2        |
|               |                |                |                |
| /boot/efi     | /boot          | root           | swap           |
|               |                | /dev/vg0/root  | /dev/vg0/swap  |
|               |                +----------------+----------------+
| /dev/sda1     | /dev/sda2      | /dev/sda3                       |
| unencrypted   | LUKS encrypted | encrypted LVM on LUKS           |
+---------------+----------------+---------------------------------+
```

### Remarks
On an SSD, instead of ext4, f2fs may be used for better performance, less journaling.

`/boot` is not required to be kept in a separate partition.
It may also be placed in the volume group of Linux LVM.


## Download 

Download the archiso [image](https://archlinux.org/download/).  
Check the hash.  
```
$ sha256sum archlinux.iso
```

Copy to a usb-drive
```
$ lsblk # find usb device
$ umount /dev/sdX
$ dd if=archlinux.iso of=/dev/sdX bs=4M status=progress && sync
```
where `sdX` is your USB device.

Boot the target computer with this usb drive.

The archlinux kernel is not signed.
Secure boot will not work by default and has to be disabled!


## Wiping

If needed securely wipe drive according this [article](https://wiki.archlinux.org/index.php/Dm-crypt/Drive_preparation).
But the step may be skipped because your partition gets encrypted anyway and it stresses the SSD.


## Preliminary steps

### Keyboard layout
If required, load another keyboard layout of your choice,
  like german layout:
```
$ loadkeys de
```

### Connect to Wi-Fi

Skip this if you have wired connection.
Just use LAN.

```
$ iwctl
iwd> device list # list devices
iwd> station <wlanX> scan # list stations
iwd> station <wlanX> get-networks # list available networks
iwd> station <wlanX> connect <SSID>
iwd> // type pw
iwd> exit

# Test connection
$ ping 8.8.8.8 -c 4
```


## Enable network time synchronization
```
$ timedatectl set-ntp true
# Check it
$ timedatectl status
```

## Partitioning

### Create partitions

Use the partition manager you like
```
$ cfdisk \dev\sdX  # easy gui like
# or
$ cgdisk /dev/sdX
$ gdisk /dev/sdX
```

Using `gdisk`
```
# Use "o" to create GPT table
# "n" to create partitions:
# Number  Start (sector)    End (sector)  Size       Code  Name
#    1            2048         1050623   1024.0 MiB  EF00  EFI System
#    2         1050624         1460223   1024.0 MiB  8300  Linux filesystem
#    3         1460224      1679181823   800.0 GiB   8E00  Linux LVM
# "w" to write changes
# "q" to quit
```

So the EFI and boot partition got 1GB each, the linux partition got the rest.

For the first two, 512MB may be enough.
Depending on the number of kernels and or dual boot setup, update frequency, 1GB is the better choice for boot.


### Initialize partitions
[`pvcreate`](https://linux.die.net/man/8/pvcreate) initializes a disk or partition as a physical volume  
[`vgcreate`](https://linux.die.net/man/8/vgcreate) creates a new volume group using block devices  
[`lvcreate`](https://linux.die.net/man/8/lvcreate) creates a new logical volume in a volume group  

The devices are named as shown in [Desired layout](#desired-layout) and may have to be adjusted to you system.
But usually it should be 
```
/dev/<something>1 (efi)
/dev/<something>2 (boot)
/dev/<something>3 (root)
```


Make filesystem for EFI. 
Commonly it's formated FAT32.
```
$ mkfs.fat -F32 /dev/sda1
```

Load crypt [modules](https://linux.die.net/man/8/modprobe)
```
$ modprobe dm-crypt
```

Create encrypted `/boot` partition
with custom options
- `-s` keysize. Default size is 256.
- `--hash` hash algorithm. Default is sha256.
- `--type` luks1 (if grub has problems with luks2, which it should not have anymore until 2020)
- `--pbkdf` pbkdf2 (default on luks2 is argon2id, but grub does not like this, so this is mandatory until now for cryptboot)
```
$ cryptsetup luksFormat -c aes-xts-plain64 -y -s 512 --hash sha512 --pbkdf pbkdf2 --iter-time 3000 /dev/sda2
$ cryptsetup open /dev/sda2 cryptboot
$ mkfs.ext2 /dev/mapper/cryptboot
```

Create encrypted LVM with `/root` and `swap`.
This one can be argon2id if you like.
```
$ cryptsetup luksFormat -c aes-xts-plain64 -y -s 512 --hash sha512 [--pbkdf pbkdf2|argon2id] --iter-time 3000 /dev/sda3
$ cryptsetup open /dev/sda3 cryptlvm
$ pvcreate /dev/mapper/cryptlvm
$ vgcreate vg0 /dev/mapper/cryptlvm
$ lvcreate -L 16G vg0 -n swap
$ lvcreate -l 100%FREE vg0 -n root
$ mkfs.ext4 /dev/mapper/vg0-root
$ mkswap /dev/mapper/vg0-swap
```

Alternativly you could put root and home in different logical volumes:
```
lvcreate -L 200GB -n root vg0
lvcreate -l 100%FREE -n home vg0
mkfs.ext4 /dev/mapper/vg0-root
mkfs.ext4 /dev/mapper/vg0-home
```


### Mount

```
$ swapon /dev/mapper/vg0-swap
$ mount /dev/mapper/vg0-root /mnt
    [$ mkdir /mnt/home
    $ mount /dev/mapper/vg0-home /mnt/home]
$ mkdir /mnt/boot
$ mount /dev/mapper/cryptboot /mnt/boot
$ mkdir /mnt/boot/efi
$ mount /dev/sda1 /mnt/boot/efi
```

### Check
```
$ lsblk
```
You will have something like this:
```text
# NAME           MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINT
# loop0            7:0    0 347.9M  1 loop  /run/archiso/sfs/airootfs
# sdb              8:32   1   3.8G  0 disk
# +-sdb2           8:34   1    40M  0 part
# +-sdb1           8:33   1   797M  0 part  /run/archiso/bootmnt
# sda              8:0    0 931.5G  0 disk
# +-sda2           8:2    0     1G  0 part
# | +-cryptboot  254:0    0     1G  0 crypt /mnt/boot
# +-sda3           8:3    0   800G  0 part
# | +-cryptlvm   254:1    0   800G  0 crypt
# |   +-vg0-swap 254:2    0    16G  0 lvm   SWAP
# |   +-vg0-root 254:3    0   784G  0 lvm   /mnt
[# |   +-vg0-root 254:3    0   784G  0 lvm   /mnt/home]
# +-sda1           8:1    0   512M  0 part  /mnt/boot/efi
```


## Install System

Uncomment far away servers or move near ones up
```
$ nano /etc/pacman.d/mirrorlist
```

Install basic modules
```
$ pacstrap /mnt base base-devel linux linux-firmware grub-efi-x86_64 vim efibootmgr wpa_supplicant lvm2
```
Use `linux-hardened` instead of `linux` if you want that.  
`vim` is not needed, if you're ok with `nano`.

Generate fstab
```
$ genfstab -pU /mnt >> /mnt/etc/fstab
```


## Init System

Chroot into our newly installed system
```
$ arch-chroot /mnt
```

Set timezone
```
$ ln -sf /usr/share/zoneinfo/Europe/Berlin /etc/localtime
$ hwclock --systohc --utc
```

Configure locales.  
This is for an english system with german sorting and time format
```
$ vim /etc/locale.conf

1> LANG=en_US.UTF-8
( use LANG=de_DE.UTF-8 for a german system )
2>
3> # Keep the default sort order (e.g. files starting with a '.'
4> # should appear at the start of a directory listing.)
5> LC_COLLATE=de_DE.UTF-8
6>
7> LC_TIME=de_DE.UTF-8
```

Edit locale.gen to enable desired languages on the system
```
$ vim /etc/locale.gen
# uncomment de_DE.UTF-8 and en_US.UTF-8 and what ever languages you may need

# and generate
locale-gen
```

Keymap and font settings

```
$ echo <awsaomeHostName> >> /etc/hostname
$ vim /etc/vconsole.conf
1> KEYMAP=de
2> FONT=lat9w-16
3> FONT_MAP=8859-1_to_uni
```

Set root password
```
$ passwd
```



Init Cpio configuration
```
$ vim /etc/mkinitcpio.conf
# and set/replace
MODULES=(ext4)
HOOKS="base udev autodetect microcode modconf kms keyboard keymap consolefont block encrypt lvm2 filesystems resume fsck"
```
- keyboard: before encrypt, to be able to use the keyboard to type key before decryption
- keymap: to load the german layout
- lvm2: before filesystem, because they are managed with lvm
- add "resume" for hibernation, if needed
- add "numlock" to enable num lock as early as possible. 
  Got an error when building: Not found!!
  So I skipped this.

Regenerate initrd image
```
$ mkinitcpio -p linux
```

If you got warnings about missing firmware for wd719x and aic94xx, etc 
  you can ignore it, if you don't have it.
You should install it from AUR if you actually use it.




## Grub
Change grub config.

`root=` This parameter specifies the device of the actual (decrypted) root file system.  
`cryptdevice=` This specifies the device containing the encrypted root on a cold boot. 
             It is parsed by the encrypt hook to identify which device contains the encrypted system.  
`cryptkey=` This parameter specifies the location of a keyfile and is required by the encrypt hook for reading such a keyfile to unlock the cryptdevice.  
`crypto=` This parameter is specific to pass dm-crypt plain mode options to the encrypt hook. 
        `crypto=hash:cipher:keysize:offset:skip`
        `crypto=sha512:aes-xts-plain64:512:0:`

```
echo "GRUB_ENABLE_CRYPTODISK=y" >> /etc/default/grub
sed -i "s#^GRUB_CMDLINE_LINUX=.*#GRUB_CMDLINE_LINUX=\"cryptdevice=UUID=$(blkid /dev/sda3 -s UUID -o value):lvm resume=/dev/mapper/vg0-swap\"#g" /etc/default/grub
```
or
```
$ vim /etc/default/grub
```
search
```
GRUB_ENABLE_CRYPTODISK=y
GRUB_CMDLINE_LINUX_DEFAULT="... intel_iommu=on"

# <root_uuid> is the uuid without brackets
GRUB_CMDLINE_LINUX="cryptdevice=UUID=<root_uuid>:cryptlvm"
# or with a swap partition to enable hibernation
GRUB_CMDLINE_LINUX="cryptdevice=UUID=<root_uuid>:cryptlvm resume=/dev/mapper/vg0-swap"
# alternatives 
# GRUB_CMDLINE_LINUX="cryptdevice=UUID=<root_uuid>:cryptlvm root=/dev/mapper/vg0-root"
# GRUB_CMDLINE_LINUX="cryptdevice=UUID=<root_uuid>:cryptlvm root=/dev/mapper/vg0-root crypto=sha512:aes-xts-plain64:512:0:"

GRUB_PRELOAD_MODULES="part_gpt part_msdos gcry_sha512"
# GRUB_PRELOAD_MODULES="part_gpt part_msdos gcry_sha512 luks cryptodisk lvm ext2"
```
`root=...` seems not to be needed, since it occurs twice `/proc/cmdline` if added here.
Unsure if this causes problems.

`<root_uuid>` is
```
$ blkid /dev/sda3 -s UUID
# in vim itself the command may be executed by
:$! blkid /dev/sda3 -s UUID

# all uuids
$ blkid -o +UUID
```

Optionally do this for a nicer grub menu layout
```
GRUB_DISABLE_SUBMENU=y
GRUB_DEFAULT=saved
GRUB_SAVEDEFAULT=true
```
This will disable submenus in grub and use the last selected kernel as the default.

Make grub config file
```
$ mkdir /boot/grub
$ grub-mkconfig -o /boot/grub/grub.cfg
```


Install grub (after configuring /etc/default/grub !!!)  
```
$ grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=ArchLinux [--modules="part_gpt part_msdos gcry_sha512"]
# on a Dell, and for boot security
$ mkdir /boot/efi/EFI/boot
$ cp /boot/efi/EFI/ArchLinux/grubx64.efi /boot/efi/EFI/boot/bootx64.efi
```
I experience issues when booting, getting the prompt for a password to display (errors regarding cryptouuid, cryptodisk, or "device not found").
I appended `--modules="part_gpt part_msdos gcry_sha512` to the end of your grub-install command.
This fixed the issues.
Just `gcry_sha512` may have been enough, though.



## Crypttab
Edit crypttab and add boot uuid
```
$ vim /etc/crypttab
# add a line
1> cryptboot /dev/sda2 none luks
or
1> cryptboot UUID=<uuidOfSda2> none luks
```

## Done

The basic setup is done.
You could jump to [Exit](exit) and check if your system boots.
Then continue with the following steps to your like.



## Additional steps

### Ease of access

For mounting boot without additional password request

```
$ dd bs=512 count=8 if=/dev/urandom of=/etc/key
$ chmod 400 /etc/key
$ cryptsetup luksAddKey --pbkdf pbkdf2 --hash=512 /dev/sda2 /etc/key
# enter any exiting passphrase: ...
```

add/replace it in crypttab
```
$ vim /etc/crypttab
1> cryptboot /dev/sda2 /etc/key luks[,discard,key-slot=1]
or
1> cryptboot UUID=<uuidOfSda2> /etc/key luks[,discard,key-slot=1]
```

Key slot is added manually to reduce boot time. 
Otherwise, all slots would be checked for a key.
Should be 1 but can be confirmed by checking
```
$ cryptsetup luksDump /dev/sda2"
```

Same thing for lvm.
Open LVM without password prompt

```
$ dd bs=512 count=8 if=/dev/urandom of=/crypto_keyfile.bin
$ chmod 000 /crypto_keyfile.bin
$ cryptsetup luksAddKey --pbkdf pbkdf2 /dev/sda3 /crypto_keyfile.bin
# enter any exiting passphrase: ...
```

Update mkinitcpio.conf
```
sed -i 's\^FILES=.*\FILES="/crypto_keyfile.bin"\g' /etc/mkinitcpio.conf
# or
vim /etc/mkinitcpio.conf
# replace/set
vim> FILES="/crypto_keyfile.bin"

# update
$ mkinitcpio -p linux
```

Update grub  
/crypto_keyfile.bin is the default name, and it would not have to be specified
  but if it doesn't work or you have chosen a different name
```
$ vim /etc/default/grub
vim> GRUB_CMDLINE_LINUX="... cryptkey=rootfs:/crypto_keyfile.bin"
$ grub-mkconfig -o /boot/grub/grub.cfg
```



### Intel microcode

On Intel machines enable Intel microcode CPU updates
```
$ pacman -S intel-ucode
```



### Some additional security
```
$ chmod 600 /boot/initramfs-linux*
$ chmod 700 /boot
$ chmod 700 /etc/iptables
```


### Additional users
Create non-root user, set password
```
$ useradd -m -g users -G wheel <username>
$ passwd <username>

# Open file
$ vim /etc/sudoers
# and uncomment string 
vim> %wheel ALL=(ALL) ALL
```


### Internet
Automatically get ip address when booting
```
$ pacman -S dhcpcd
$ systemctl enable dhcpcd.service
```



### Dual Boot
Skip this on a single boot system.

Install os-prober if dual boot is considered to detect other OSes.
Using it is considered a security risk by grub itself.
So you may don't want to use it and manually add the other system.  
(untested)
```
$ pacman -S os-prober
$ vim /etc/default/grub
# add/comment line
GRUB_DISABLE_OS_PROBER=false

# or do it manually [more secure]
# get <uuid> of windows partition
$ blkid /dev/sda2 -s UUID -o value
$ vim /etc/grub.d/40_custom
# add menu entry at the end
## uefi
menuentry "Windows 10" {
    insmod ntfs
    insmod chain
    search -u uuid-of-windows-esp
    chainloader /EFI/Microsoft/Boot/bootmgfw.efi
}
```


Make grub config file
```
$ mkdir /boot/grub
$ grub-mkconfig -o /boot/grub/grub.cfg
```

## Finish

Exit from chroot, unmount system, shutdown, extract flash stick. You made it! Now you have fully encrypted system.
$ exit
$ umount -R /mnt
$ swapoff -a
$ poweroff


## Use your system
Reboot again, login as user, install all other software you like: 
drivers, display server, desktop environment, etc...
- https://wiki.archlinux.de/title/Anleitung_f%C3%BCr_Einsteiger#Weitere_n.C3.BCtzliche_Dienste
- https://wiki.archlinux.de/title/Anleitung_f%C3%BCr_Einsteiger#Teil_2:_Installation_von_X_und_Konfiguration



## Aux

### Secure boot

untested:

The archlinux kernel is not signed.
Secure boot will not work by default and has to be disabled.

For additional security, start PC, login in UEFI menu during boot (in most cases by pressing F2 or DEL)
Enable Secure boot option, and choose our EFI image as trusted. 
Path will be something like this:
HDD0 -> EFI -> ArchLinux -> grubx64.efi

Of course, you must protect the UEFI menu with a password.
Choose DIFFERENT password from that you used for encryption, because some lazy manufacturers store this password not securely enough.


### Temp Device
```
# $ vim /etc/fstab
# add line
> tempfs  /home/{username}/temp   tempfs  mode=0777,noatime   0   0
```

encrypt  
```
# $ vim /etc/crypttab
...
```



## Troubleshooting

### Operation system not found
https://bbs.archlinux.org/viewtopic.php?id=226003
Ok - solved it!
Needed to unset the bootable flag from /dev/sda1 (where grub installed itself) and make it /dev/sda2 (where the operating system actually is).

https://superuser.com/questions/645940/syslinux-missing-os-after-expanding-to-the-left-the-partition-with-boot
arch-chroot into the drive after mounting the partitions and then installing the bootloader.


### lvmetad.socket
If you got errors "/run/lvm/lvmetad.socket: connect failed: No such file or directory", that's OK.
You can get rid of this errors with some workarounds, but this is not really necessary.
In any case DO NOT disable lvmetad! 

https://wiki.archlinux.org/index.php/GRUB#Warning_when_installing_in_chroot



### efivars/efivarfs
https://unix.stackexchange.com/questions/91620/efi-variables-are-not-supported-on-this-system  
https://bbs.archlinux.org/viewtopic.php?id=172867  
https://wiki.archlinux.org/index.php/Unified_Extensible_Firmware_Interface#Mount_efivarfs  



### Not Bootable

#### Grub and Dell
https://wiki.archlinux.org/index.php/GRUB/Tips_and_tricks#UEFI_firmware_workaround

esp is the UEFI partition mountpoint, i.e. /boot/efi/ in this tutorial
```
$ mkdir esp/EFI/boot
$ cp esp/EFI/grub_uefi/grubx64.efi esp/EFI/boot/bootx64.efi
i.e.:
$ mkdir /boot/EFI/boot/
$ cp boot/efi/ArchLinux/grubx64.efi /boot/EFI/boot/bootx64.efi
$ reboot
```

#### Missing Modules
https://www.linux.org/threads/understanding-the-various-grub-modules.11142/

If password invalid maybe some modules are missing
```
grub> insmod <modulename>
```


### Grub and German Keyboard

Important if boot crypto password contains special chars  
https://askubuntu.com/questions/751259/how-to-change-grub-command-line-grub-shell-keyboard-layout
https://superuser.com/a/1188564


### Grub cmd
Typed wrong pw for (hd0,gpt2).
Then boot manually by
```
grub> cryptomount (hd0,gpt2)
type pw
grub> insmod normal
grub> normal
```



### No Hibernation Device found 

Failed to start file system check on /dev/disk/by-uuid

Get full error in emergency shell
```
uuid= 06618d7a-0b70-4ccd-9953-d5deae8c8e009
systemctl status --no-pager --full systemd-fsck@dev-disk-by\\x2duuid-06618d7a\\x2d0b70\\x2d4ccd\\x2d9953\\x2d5deae8c8e009.service

fsck -f /dev/sda2
fsck -v /dev/sdXx
```

### Battery
Check capacity in terminal
```
$ cat /sys/class/power_supply/BAT0/capacity
```


### Stuff

cat /proc/cmdline


## Recovery
If the system does not boot, 
  you can always try to recover it by booting in with an archlinux installation medium.

Boot with arch linux usb.  
To find out device layout, type:
```
$ fdisk -l
or
$ lsblk
```

In this case encrypted boot is sda2 and lvm is sda3:
```
# cryptsetup <cmd> <device> <name>
$ cryptsetup open /dev/sda2 cryptboot
$ cryptsetup open /dev/sda3 cryptlvm

$ swapon /dev/mapper/vg0-swap
$ mount /dev/mapper/vg0-root /mnt
$ mount /dev/mapper/cryptboot /mnt/boot
$ mount /dev/sda1 /mnt/boot/efi

$ arch-chroot /mnt

arch> <do your stuff>

$ exit
$ swapoff -a
$ umount -R /mnt
[$ cryptsetup close vg0]
[$ cryptsetup close cryptboot]
$ poweroff
```

## Keys
Change pw in slot 0
```
$ cryptsetup luksChangeKey /dev/sda2 -S 0 [--pbkdf pbkdf2 --iter-time 3000 --hash sha512]
```

Delete pw in slot 2
```
$ cryptsetup luksKillSlot /dev/sda2 2
```

Add key to slot 5
```
$ cryptsetup luksAddKey /dev/sda2 -S 5 [--pbkdf pbkdf2 --iter-time 3000 --hash sha512]
```

Check key slots, header
```
$ cryptsetup luksDump /dev/sda2
```

## LVM

### Volume Group Info
```
vgs
vgdisplay
```

### Logical Volume Info
```
lvs
lvdisplay
```

### Resize Logical Volume
```
lvresize --size [+|-]LogicalVolumeSize[bBsSkKmMgGtTpPeE] [--resizefs] <lvname>
```
If the logical volume is formatted ext4 for example, `--resizefs` is required to resize the underlying filesystem.

https://linux.die.net/man/8/lvresize


## Kernels
You can install different kernels.
Just install the kernel of your choice and regenerate grub config.
```
$ pacman -S linux-hardened
$ grub-mkconfig -o /boot/grub/grub.cfg
```


## Links

### Installation
- https://wiki.archlinux.de/title/Systemverschl%C3%BCsselung_mit_dm-crypt

### Crypto
- https://cryptsetup-team.pages.debian.net/cryptsetup/encrypted-boot.html
- https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system#Encrypted_boot_partition_(GRUB)
- https://wiki.archlinux.org/title/Dm-crypt/System_configuration#mkinitcpio
- https://wiki.archlinux.org/title/GRUB#LUKS2


