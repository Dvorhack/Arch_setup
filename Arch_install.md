# Arch Linux on Asus ROG Zephyrus G14 (G401II)
My own notes installing Arch Linux with btrfs, disc encryption, auto-snapshots, no-noise fan-curves on my Asus ROG ROG


## Basic Install

### Prepare and Booting ISO

Change keyboard layout with  `loadkeys fr`

### Networking

For Network i use wireless, if you need wired please check the [Arch WiKi](https://wiki.archlinux.org/index.php/Network_configuration).

Launch `iwctl` and connect to your AP
```bash
iwctl
> device list
> station [device_name] scan
> station [device_name] get-networks
> station [device_name] connect SSID
> exit
```

Update System clock with `timedatectl set-timezone Europe/Paris`

### Format Disk

* My Disk is `nvme0n1`, check with `lsblk`
* Format Disk using `gdisk /dev/nvme0n1` with this simple layout:

	* `o` for new partition table
	* `n,1,<ENTER>,+1024M,ef00` for EFI Boot
	* `n,2,<ENTER>,<ENTER>,8300` for the linux partition
	* `w` to save layout


Format the EFI Partition

`mkfs.vfat -F 32 -n EFI /dev/nvme0n1p1`

### Create encrypted filesystem

You can name your decrypted volume as you want. I'll refer to **$VolName** so you just have to replace it with anything.

```bash
cryptsetup luksFormat /dev/nvme0n1p2
cryptsetup open /dev/nvme0n1p2 $VolName
```

### Create and Mount btrfs Subvolumes

Format the btrfs partition
`mkfs.btrfs -f -L ROOTFS /dev/mapper/$VolName`


Mount Partitions und create Subvol for btrfs. I dont want home, etc in my snapshots, so create subvol for them.

* `mount -t btrfs LABEL=ROOTFS /mnt` Mount root filesystem to /mnt
* `btrfs sub create /mnt/@`
* `btrfs sub create /mnt/@home`
* `btrfs sub create /mnt/@snapshots`
* `btrfs sub create /mnt/@swap`

### Create a btrfs swapfile and remount subvols

```bash
truncate -s 0 /mnt/@swap/swapfile
chattr +C /mnt/@swap/swapfile
fallocate -l 16G /mnt/@swap/swapfile
chmod 600 /mnt/@swap/swapfile
mkswap /mnt/@swap/swapfile
mkdir /mnt/@/swap
```

Just unmount with `umount /mnt/` and remount with subvolumes

```bash
mount -o noatime,compress=zstd,commit=120,subvol=@ /dev/mapper/$VolName /mnt
mkdir -p /mnt/boot
mkdir -p /mnt/home
mkdir -p /mnt/.snapshots
mkdir -p /mnt/btrfs

mount -o noatime,compress=zstd,commit=120,subvol=@home /dev/mapper/$VolName /mnt/home/
mount -o noatime,compress=zstd,commit=120,subvol=@snapshots /dev/mapper/$VolName /mnt/.snapshots/
mount -o noatime,commit=120,subvol=@swap /dev/mapper/$VolName /mnt/swap/

mount /dev/nvme0n1p1 /mnt/boot/

mount -o noatime,compress=zstd,commit=120,subvolid=5 /dev/mapper/$VolName /mnt/btrfs/
```


Check mountmoints with `df -Th`

### Install the system using pacstrap

```bash
pacstrap /mnt base base-devel linux linux-firmware btrfs-progs nano vi networkmanager amd-ucode
```

After this, generate the filesystem table using
`genfstab -Lp /mnt >> /mnt/etc/fstab`

Add swapfile
`echo "/swap/swapfile none swap defaults 0 0" >> /mnt/etc/fstab `

### Chroot into the new system and change language settings

```bash
arch-chroot /mnt
echo {MYHOSTNAME} > /etc/hostname
echo LANG=fr_FR.UTF-8 > /etc/locale.conf
echo LANGUAGE=fr_FR >> /etc/locale.conf
echo KEYMAP=fr > /etc/vconsole.conf
echo FONT=lat9w-16 >> /etc/vconsole.conf
ln -sf /usr/share/zoneinfo/Europe/Paris /etc/localtime
```

Modify `nano /etc/hosts` with these entries. For static IPs, remove 127.0.1.1

```bash
127.0.0.1		localhost
::1				localhost
127.0.1.1		{MYHOSTNAME}.localdomain	{MYHOSTNAME}
```

`nano /etc/locale.gen` to uncomment the following lines

```bash
fr_FR.UTF-8 UTF-8
fr_FR ISO-8859-1
fr_FR@euro ISO-8859-15
en_US.UTF-8
```
Execute `locale-gen` to create the locales now

Add a password for root using `passwd root`


### Add btrfs and encrypt to Initramfs

`nano /etc/mkinitcpio.conf` and add **encrypt btrfs** to hooks between block/filesystems

It might look like this  
`HOOKS="base udev autodetect modconf block encrypt btrfs filesystems keyboard fsck `

Also include **amdgpu** in the MODULES section

create Initramfs using `mkinitcpio -p linux`

### Install Systemd Bootloader

`bootctl --path=/boot install` installs bootloader

`nano /boot/loader/loader.conf` delete anything and add these few lines and save

```bash
default	arch.conf
timeout	3
editor	0
```

` nano /boot/loader/entries/arch.conf` with these lines and save.

```bash
title	Arch Linux
linux	/vmlinuz-linux
initrd	/amd-ucode.img
initrd	/initramfs-linux.img
```

copy boot-options with
` echo "options	cryptdevice=UUID=$(blkid -s UUID -o value /dev/nvme0n1p2):$VolName root=/dev/mapper/$VolName rootflags=subvol=@ rw" >> /boot/loader/entries/arch.conf`

### Set nvidia-nouveau onto blacklist

using `nano /etc/modprobe.d/blacklist-nvidia-nouveau.conf` with these lines

```bash
	blacklist nouveau
	options nouveau modeset=0
```

### Leave Chroot and Reboot

Type `exit` to exit chroot

`umount -R /mnt/` to unmount all volumes

Now its time to `reboot` into the new system!

## Finetuning after first Reboot

### Enable Networkmanager

Configure WiFi Connection.

```bash
systemctl enable --now NetworkManager
nmcli device wifi connect "{YOURSSID}" password "{SSIDPASSWORD}"
```

### Enable NTP Timeservice

```bash
systemctl enable --now systemd-timesyncd.service
```

(You may look at `/etc/systemd/timesyncd.conf` for default values and change if necessary)

### Create a new user

First create a new local user and point it to bash

```bash
groupadd sudo
useradd -m -G sudo,lp,power,audio -s /bin/bash {MYUSERNAME}
passwd {MYUSERNAME}
```

Execute `visudo` and uncomment `%sudo ALL=(ALL) ALL`

Now exit (`:wq`) and relogin with the new {MYUSERNAME}


Install some Deamons before we reboot

```bash
sudo pacman -Sy acpid dbus
sudo systemctl enable acpid
```

## Setup automatic Snapshots for Pacman

The goal is to have automatic snapshots each time i made changes with pacman. The hook creates a snapshot
to ".snapshots\STABLE". So if something goes wrong i can boot from this snapshot and rollback my system.


### Create the STABLE snapshot and modify Bootloader

First i create the snapshot and changes by hand to test if anythink is working. After that it will be done automaticly by our hook and script.

```bash
sudo -i
btrfs sub snap / /.snapshots/STABLE
cp /boot/vmlinuz-linux /boot/vmlinuz-linux-stable
cp /boot/amd-ucode.img /boot/amd-ucode-stable.img
cp /boot/initramfs-linux.img /boot/initramfs-linux-stable.img
cp /boot/loader/entries/arch.conf /boot/loader/entries/stable.conf
```

Edit `/boot/loader/entries/stable.conf` to boot from STABLE snapshot

```bash
title   Arch Linux Stable
linux   /vmlinuz-linux-stable
initrd  /amd-ucode-stable.img
initrd  /initramfs-linux-stable.img
options ... rootflags=subvol=@snapshots/STABLE rw
```

Now edit the `/.snapshots/STABLE/etc/fstab` to change the root to the new snapshot/STABLE

ˋˋˋ
...
LABEL=ROOTFS  /  btrfs  rw,noatime,.....subvol=@snapshots/STABLE
...
ˋˋˋ

reboot and test if you can boot from the stable snapshot.

### Script for auto-snapshots

Copy the [script](res/autosnap) from my Repo to

`/usr/bin/autosnap` and make it executable with `chmod +x /usr/bin/autosnap`

### Hook for pacman

Copy the [script](res/00-autosnap.hook) from my Repo to

`/etc/pacman.d/hooks/00-autosnap.hook`

Now each time pacman executes, it launches the `autosnap`script which takes a snapshot from the current system.



## Install Desktop Environment

### Get X.Org and i3

Install xorg and i3 packages

```bash
sudo pacman -Sy xorg i3 i3wm xf86-input-synaptics gvfs xdg-user-dirs ttf-dejavu pulseaudio network-manager-applet firefox-i18n-fr git git-lfs curl wget

sudo localectl set-x11-keymap fr
xdg-user-dirs-update
```

LightDM Loginmanager

```bash
sudo pacman -S lightdm lightdm-gtk-greeter lightdm-gtk-greeter-settings
sudo systemctl enable lightdm
```

Reboot and login to your new Desktop. For i3 customization, you can follow my [i3 config](i3_config.md)


### Oh-My-ZSH

I like to use oh-my-zsh with Powerlevel10K theme

```bash
sudo pacman -Sy zsh zsh-completions
chsh -s /bin/zsh # (relogin to activate)
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
cd ~/.local/share/fonts
wget https://github.com/romkatv/powerlevel10k-media/raw/master/MesloLGS%20NF%20Regular.ttf
wget https://github.com/romkatv/powerlevel10k-media/raw/master/MesloLGS%20NF%20Bold.ttf
wget https://github.com/romkatv/powerlevel10k-media/raw/master/MesloLGS%20NF%20Italic.ttf
wget https://github.com/romkatv/powerlevel10k-media/raw/master/MesloLGS%20NF%20Bold%20Italic.ttf
fc-cache -v
git clone --depth=1 https://github.com/romkatv/powerlevel10k.git ${ZSH_CUSTOM:-$HOME/.oh-my-zsh/custom}/themes/powerlevel10k
```

Set `ZSH_THEME="powerlevel10k/powerlevel10k"` in `~/.zshrc`

### Setup Plymouth for nice Password Prompt during Boot

Plymouth is in [AUR](https://aur.archlinux.org) , just clone the Repo and make the Package (i create a subfolder AUR within my homefolder)

```bash
cd AUR
git clone https://aur.archlinux.org/plymouth-git.git
makepkg -is
```

Now modify the Hooks for the Initramfs using `/etc/mkinitcpio.conf`, Plymouth must be right after "base udev". Delete encrypt hook, it will be replaced by plymouth-encrypt

```bash
HOOKS="base udev plymouth plymouth-encrypt autodetect modconf block btrfs filesystems keyboard fsck
```

Run `mkinitcpio -p linux`

Add some kernel-parameters to make boot smooth. Edit `/boot/loader/entries/arch.conf` and append to options

```bash
...rootflags=subvol=@ quiet splash loglevel=3 rd.systemd.show_status=auto rd.udev.log_priority=3 vt.global_cursor_default=0 rw
```

Optional: replace LightDM with the plymouth variant

```bash
sudo systemctl disable lightdm
sudo systemctl enable lightdm-plymouth
```

For Plymouth Theming and Options, check [Plymouth on Arch Wiki](https://wiki.archlinux.org/title/plymouth)


## Nvidia Driver

### Install latest nvidia driver

I'am going to use the nvidia-dkms package, as i will use a custom kernel

```bash
sudo pacman -Sy nvidia-dkms nvidia-settings nvidia-prime acpi_call linux-headers
```

Reboot after install.

## Tweaks

### Add Custom Repo from [Luke Jones](https://asus-linux.org/) and install some tools
This part is need only for people with Asus gaming laptops (rog, zephyrus, ...)

#### Add G14 Repository

```bash
sudo bash -c "echo -e '\r[g14]\nSigLevel = DatabaseNever Optional TrustAll\nServer = https://arch.asus-linux.org\n' >> /etc/pacman.conf"

sudo pacman -Sy asusctl supergfxctl
sudo systemctl enable --now power-profiles-daemon.service
sudo systemctl enable --now supergfxd
```

#### Install custom Kernel and change bootloader

```bash
sudo pacman -Sy linux-g14 linux-g14-headers
sudo cp /boot/loader/entries/arch.conf /boot/loader/entries/arch_g14.conf
sudo sed -i 's/Arch Linux/Arch Linux G14/' /boot/loader/entries/arch_g14.conf
sudo sed -i 's/vmlinuz-linux/vmlinuz-linux-g14/' /boot/loader/entries/arch_g14.conf
sudo sed -i 's/initramfs-linux/initramfs-linux-g14/' /boot/loader/entries/arch_g14.conf
```

If you want the autosnap to save the G14 enironment, youneed to modify `autosnap` accordingly.

Reboot now!

After reboot you can check available features with `asusctl -s` . Fan curves, profiles should be possible with custom kernel.

#### Modify profiles for asusctl

You can use [my file](res/profiles.conf) from `etc/asusd/profiles.conf`. But its better to delete the existing file, after reboot its regenerated with default and read-out values, The file is self -descriptive.

### Setup Bluetooth

Install required bluetooth modules and add your user to the group `lp`
```bash
sudo pacman -Sy bluez bluez-utils blueman
sudo systemctl enable --now bluethooth.service
```

### Enable Autologin

Since iam running full disc encryption, i would like to enable autologin to my X-Session. Open `/etc/lightdm/lightdm.conf` and add theese under `[Seat:*]`

```bash
pam-service=lightdm
pam-autologin-service=lightdm-autologin
autologin-user={MYUSERNAME}
autologin-user-timeout=0
session-wrapper=/etc/lightdm/Xsession
greeter-session=lightdm-greeter

```

Now create the group autologin and add your user

```bash
sudo groupadd -r autologin
sudo gpasswd -a {MYUSERNAME} autologin
```

