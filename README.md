# Arch Linux Installation Cheat Sheet (T460p Optimized)

inspired by [ML4W](https://www.ml4w.com)

I recently upgraded my laptop with a new 1TB SSD and 16GB RAM.
With an Intel i5-6440HQ CPU and Intel HD 530 GPU, I decided to reinstall a
clean, modern, and optimized Arch Linux system (minimal but not that "minimal") . This guide is intended for
programming and school use, and can serve as a reference for others(hoping). I chose to
migrate from ext4 to Btrfs for the features.

I opted not to use encryption due to the inconvenience of double login screens.
I plan to set up encryption once I have a laptop with TPM support (hopping for a framework13), allowing for
a single themed login screen. Your advice and insights are welcome as I want to
learn and improve.

This guide is based on extensive documentation, "AI" prompting, and video tutorials.
There are still areas for improvement and further study, such as Btrfs swap
configuration ( see files).

## Preparation

```bash
loadkeys us
timedatectl set-ntp true
iwctl --passphrase [password] station wlan0 connect [SSID]
ping -c4 archlinux.org
```

## Partition SATA SSD (/dev/sda)

```bash
gdisk /dev/sda
# 1: +512M (ef00 - EFI)
# 2: Remaining space (8300 - Linux FS)
# Write (w), Confirm (Y)
```

## Btrfs Configuration (Optimized)

```bash
mkfs.fat -F32 /dev/sda1
mkfs.btrfs -f -L ARCH /dev/sda2
```

## Create subvolumes

```bash
mount /dev/sda2 /mnt
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@snapshots
btrfs subvolume create /mnt/@var
btrfs subvolume create /mnt/@var_log
btrfs subvolume create /mnt/@var_cache
umount /mnt
```

## Mount root with SSD optimizations

```bash
mount -o compress=zstd:3,noatime,space_cache=v2,discard=async,subvol=@ /dev/sda2 /mnt
```

## Create directory structure

```bash
mkdir -p /mnt/{boot/efi,home,.snapshots,var/{log,cache}}
```

## Mount subvolumes

```bash
mount -o compress=zstd:3,noatime,space_cache=v2,discard=async,subvol=@home /dev/sda2 /mnt/home
mount -o compress=zstd:3,noatime,space_cache=v2,discard=async,subvol=@snapshots /dev/sda2 /mnt/.snapshots
mount -o compress=zstd:3,noatime,space_cache=v2,discard=async,subvol=@var /dev/sda2 /mnt/var
mount -o compress=zstd:3,noatime,space_cache=v2,discard=async,subvol=@var_log /dev/sda2 /mnt/var/log
mount -o compress=zstd:3,noatime,space_cache=v2,discard=async,subvol=@var_cache /dev/sda2 /mnt/var/cache
mount /dev/sda1 /mnt/boot/efi
```

## Base Installation

```bash
pacstrap -K /mnt base base-devel linux linux-firmware intel-ucode git vim
```

## Generate fstab

```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

## System Configuration

```bash
arch-chroot /mnt

# Time & Locale
ln -sf /usr/share/zoneinfo/[region]/[city] /etc/localtime
hwclock --systohc
echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen
locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf
echo "KEYMAP=us" > /etc/vconsole.conf

```

## Essential Packages

```bash
pacman -S --noconfirm grub efibootmgr networkmanager reflector \
          pipewire wireplumber xdg-desktop-portal-hyprland \
          intel-media-driver vulkan-intel  sddm relfector bash-conpletion firewalld \
          pacman-contrib acpid brightnessctl linux-header  btop yazi zsh\
          fzf neovim openssh xdg-user-dirs acpid\

```

## Swap Configuration (4GB)

(or I make a separate partition ?)

```bash
dd if=/dev/zero of=/swapfile bs=1M count=4096
chmod 600 /swapfile
mkswap /swapfile
echo "/swapfile none swap defaults,discard 0 0" >> /etc/fstab
```

## GPU & Boot

> [!warning] Specific to my cPU

```bash
# Intel GPU configuration
echo "options i915 enable_guc=3" > /etc/modprobe.d/i915.conf
echo "MODULES=(i915)" >> /etc/mkinitcpio.conf
```

## GRUB installation

> [!warning] specific to my CPU 

```bash
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=GRUB
sed -i 's/GRUB_CMDLINE_LINUX=""/GRUB_CMDLINE_LINUX="intel_iommu=on i915.enable_guc=3"/' /etc/default/grub
grub-mkconfig -o /boot/grub/grub.cfg
```

## Regenerate initramfs

```bash
# Adding btrfs and setfont to mkinitcpio
# Before: BINARIES=()
# After: BINARIES=(btrfs setfont)

sed -i 's/BINARIES=()/BINARIES=(btrfs)/g' /etc/mkinitcpio.conf
mkinitcpio -p linux
```

```bash
mkinitcpio -P
```

## Final Steps

```zsh
# User setup
useradd -m -G wheel [USERNAME]
passwd [USERNAME]
EDITOR=vim visudo  # Uncomment %wheel line

# Enable services
systemctl enable NetworkManager fstrim.timer reflector.timer sshd bluetooth acpid

exit
umount -R /mnt
reboot

```
