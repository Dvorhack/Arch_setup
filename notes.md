## Get the iso ready

Based on: https://wiki.archlinux.org/title/Installation_guide

bases
```bash
loadkeys fr
timedatectl set-timezone Europe/Paris
```

https://wiki.archlinux.org/title/Iwd#iwctl
Connect wifi
```bash
iwctl
> device list
> station [device_name] scan
> station [device_name] get-networks
> station [device_name] connect SSID
```

## Partition the disk

List the available disks `fdisk -l`

```bash
parted /dev/sdaX # can also be /dev/nvme1nX
# on supprime les partition existantes
(parted) print
(parted) rm 1
(parted) rm 2

# création partition EFI
(parted) mkpart "EFI system partition" fat32 1Mib 513Mib
(parted) set 1 est on # On rend la partition bootable

# Création partition principale
(parted) mkpart "main partition" btrfs 513MiB 100%
```

Format EFI: `mkfs.vfat -F32 /dev/nvme0n1p1`  
Create the encrypted partition and format it 
```bash
cryptsetup -c=aes-xts-plain64 --key-size=512 --hash=sha512 --iter-time=3000 --pbkdf=pbkdf2 --use-random luksFormat --type=luks1 /dev/sdX2  

cryptsetup luksOpen /dev/sdX2 MainPart # it will open the encrypted partition
mkfs.btrfs -L "Arch Linux" /dev/mapper/MainPart  
```
Create and mount btrfs subvolumes
```bash
mount /dev/mapper/MainPart /mnt

btrfs su cr /mnt/@  
btrfs su cr /mnt/@home  
btrfs su cr /mnt/@varlog
btrfs su cr /mnt/@tmp  
btrfs su cr /mnt/@snapshots  

chattr +C /mnt/@varlog
chattr +C /mnt/@tmp  
umount /mnt  

mount -o defaults,noatime,discard,ssd,subvol=@ /dev/sdX2 /mnt  
mkdir /mnt/home  
mkdir -p /mnt/var/log  
mkdir /mnt/tmp  
mkdir /mnt/snapshots  
mkdir /mnt/efi # for EFI partition /dev/sdX1  

mount -o defaults,noatime,discard,ssd,subvol=@home /dev/mapper/MainPart /mnt/home
mount -o defaults,noatime,discard,ssd,subvol=@varlog /dev/mapper/MainPart /mnt/var/log
mount -o defaults,noatime,discard,ssd,subvol=@tmp /dev/mapper/MainPart /mnt/tmp
mount -o defaults,noatime,discard,ssd,subvol=@snapshots /dev/mapper/MainPart /mnt/snapshots

mount /dev/nvme0p1n2 /mnt/efi
```

## Install Arch
`pacstrap /mnt/ base base-devel git btrfs-progs efibootmgr linux linux-headers linux-firmware mkinitcpio dhcpcd bash-completion sudo`*

chroot into it: `arch-chroot /mnt /bin/bash`

```bash
ln -sf /usr/share/zoneinfo/Region/City /etc/localtime
hwclock --systohc

# uncomment fr-FR.UTF-8 in /etc/locale.gen
locale-gen

# Write
LANG=en_US.UTF-8
LC_COLLATE=pl_PL.UTF-8
LC_MEASUREMENT=pl_PL.UTF-8
LC_MONETARY=pl_PL.UTF-8
LC_NUMERIC=pl_PL.UTF-8
LC_TIME=pl_PL.UTF-8
# in /etc/locale.conf

# Write hostname in /etc/hostname
```

Swap file for snapshots
```bash
btrfs su create /swap

chattr +C /swap

touch /swap/swapfile  
dd if=/dev/zero of=/swap/swapfile bs=1024K count=4096  

chmod 600 /swap/swapfile  
mkswap /swap/swapfile  
swapon /swap/swapfile  

echo "/swap/swapfile none swap sw 0 0  " >> /etc/fstab
```

Passwords
```bash
passwd

useradd -m paul
passwd paul
```

Generate new mkinitcpio
edit `/etc/mkinitcpio.conf`
add `encrypt btrfs` before filesystems in HOOKS
add `btrfsck` in BINARIES
regenerate mkinitcpio `mkinitcpio -P`

## Install bootloader
```bash
grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=GRUB  
grub-mkconfig -o /boot/grub/grub.cfg  
```
ressources: https://github.com/Szwendacz99/Arch-install-encrypted-btrfs