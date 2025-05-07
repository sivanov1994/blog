---
title: "How to Install Arch Linux with Full Disk Encryption and LVM Using systemd-boot"
date: 2025-05-05
description: "A complete step-by-step guide to installing Arch Linux with full disk encryption (LUKS), LVM, and systemd-boot across two NVMe drives."
tags: ["arch linux", "encryption", "luks", "lvm", "systemd-boot", "installation"]
categories: ["linux", "security"]
draft: false
---
This guide describes how to install Arch Linux with full disk encryption, Logical Volume Management (LVM), and the minimalist systemd-boot bootloader. The setup uses two NVMe drives, as this reflects my specific hardware configuration. If you're using only one drive, the process remains mostly the same—just adapt the LUKS and LVM steps accordingly.

Let’s get started.

## Hardware for this Guide

- CPU: AMD Ryzen 9 5900X
- GPU: AMD Radeon RX 6900 XT
- Memory: 32GB
- Two NVMe drives, each 1TB  
- UEFI-enabled system (BIOS must support UEFI)  

## Preparing the Terrain

Before embarking on the installation, you’ll need a bootable USB drive with Arch Linux.

Download the latest ISO from the official [Arch Linux website](https://archlinux.org/download/) and flash it onto your USB stick. Once plugged in, identify its device path using the lsblk command:

`lsblk`

Here’s how the output looked on my machine:

```
NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sdb           8:16   1 14.9G  0 disk 
├─sdb1        8:17   1  650M  0 part /run/media/user/ARCH_202505
├─sdb2        8:18   1   64M  0 part 
└─sdb3        8:19   1  300K  0 part 
```

My USB drive is 16GB and recognized as /dev/sdb. Now it’s time to write the ISO image onto it using dd. Caution: this command will irrevocably erase all data on the drive, so double-check the device path before proceeding.

`sudo dd bs=4M if=/path/to/archlinux.iso of=/dev/sdb status=progress oflag=sync`

Wait for the process to complete — it may take a few minutes. Once it’s done, you’re ready to boot from the USB drive and dive into the real journey.

At this point we should be in front of a prompt:

`root@archiso ~ #`

This is our root prompt and I’ll be shortening that to $ in this post.

This is an OPTIONAL step but if the console font is too small or not readable, we can set it up:

`setfont latarcyrheb-sun32`

We need an internet connection, so let’s configure the network. I connected via ethernet so everything worked out of the box. If you need WiFi, you can set it up by launching iwctl (interaction mode with autocompletion). Here are some useful commands:

```bash
 iwctl
 station list                        # Display your wifi stations
 station <INTERFACE> scan            # Start looking for networks with a station
 station <INTERFACE> get-networks    # Display the networks found by a station
 station <INTERFACE> connect <SSID>  # Connect to a network with a station
```
We also need to update our system clock. Let’s use timedatectl(1) to ensure the system clock is accurate:

`timedatectl set-ntp true`

To check the service status, we can use timedatectl status.

 I tend to use this personal naming convention to identify my hardware with different operating systems, you will see this name around when setting things up in a few places, especially when setting up the volumes in LVM. Swap it out for your own.

### Disk Partitioning

Once that’s done, we can begin building up to the installation.  
Everything except the `/boot` partition will be encrypted, ensuring a secure and flexible system layout for power users who value control and privacy.

Here is my actual disk layout (run `lsblk` to inspect yours):

**Note:** The `/home` logical volume spans both disks, so its device-mapper entry may appear under both `rion` and `cryptlvm` in `lsblk`.

```
NAME            MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINTS
nvme0n1         259:0    0 931.5G  0 disk
├─nvme0n1p1     259:2    0     1G  0 part  /boot
└─nvme0n1p2     259:3    0 930.5G  0 part
  └─rion        253:0    0 930.5G  0 crypt
    ├─rion-swap 253:2    0     8G  0 lvm   [SWAP]
    ├─rion-root 253:3    0    32G  0 lvm   /
    └─rion-home 253:4    0   1.8T  0 lvm   /home
nvme1n1         259:1    0 931.5G  0 disk
└─nvme1n1p1     259:4    0 931.5G  0 part
  └─cryptlvm    253:1    0 931.5G  0 crypt
    └─rion-home 253:4    0   1.8T  0 lvm   /home
```

**Note:** The volume group `rion` spans across both encrypted disks. The root (`/`), swap, and part of the home volume reside on the first device (`rion`, from `nvme0n1p2`). The second device (`cryptlvm`, from `nvme1n1p1`) is added entirely to extend the `/home` volume.

This setup results in full disk encryption (FDE), with the sole exception of the `/boot` partition, which remains unencrypted to allow the system to boot.

For partitioning, I used the `gdisk` utility to create and manage the GPT layout:


`gdisk /dev/nvme0n1`

You’ll enter the interactive gdisk prompt. Follow these steps:

1. Create GPT label (if asked)
If the disk is empty or has no GPT, you may be prompted to create one. Type:
```bash
o
```

Confirm with `y` to create a new empty GPT.

2. Create EFI system partition (1 GB for /boot)

```bash
n
Partition number: 1
First sector: (press Enter for default)
Last sector: +1G
Hex code or GUID: ef00
```

3. Create encrypted LUKS container (rest of the disk)
```bash
n
Partition number: 2
First sector: (press Enter)
Last sector: (press Enter to use remaining space)
Hex code or GUID: 8e00 
```

I prefer to partition the second (nvme1n1)disk (e.g. for clarity or better tooling support), you can create a single GPT partition that spans the entire disk and set its hex code to 8e00 .

Steps:
```bash

gdisk /dev/nvme1n1
```
Then inside gdisk:
```bash
o                # create new GPT if needed
n                # new partition
[Enter]          # default partition number
[Enter]          # default start
[Enter]          # default end (use full disk)
8e00             # Linux LVM filesystem (used for LUKS)
w                # write and exit
```

### Setting Up Disk Encryption

Now that our partitions are in place, it’s time to secure the system. We’ll encrypt two devices: the second partition of the first disk (/dev/nvme0n1p2) and the entirety of the second disk (/dev/nvme1n1p1). These will be combined into a single LVM volume group that will host our root, swap, and home volumes — all protected by encryption.

Encrypt nvme0n1p2 (Arch system volume):
```bash

cryptsetup luksFormat /dev/nvme0n1p2
```

You'll be prompted with:
```
WARNING!
========
This will overwrite data on /dev/nvme0n1p2 irrevocably.

Are you sure? (Type uppercase yes): YES
Enter passphrase:
Verify passphrase:
```

Then open it:
```bash
cryptsetup open /dev/nvme0n1p2 rion
```

Encrypt nvme1n1p1:
```bash
cryptsetup luksFormat /dev/nvme1n1p1
```

Unlock it:
```bash
cryptsetup open /dev/nvme1n1p1 cryptlvm
```

### Setting Up LVM

Now we’ll create one logical volume group (cryptlvm) across both unlocked encrypted containers. Inside it, we’ll define three logical volumes: one for root, one for swap, and the rest for home.

```bash
pvcreate /dev/mapper/rion /dev/mapper/cryptlvm
vgcreate rion /dev/mapper/rion /dev/mapper/cryptlvm
```

**Note:** The names rion and cryptlvm used in this guide are just examples. You can choose whatever naming convention you prefer — just make sure to use the same name consistently throughout the setup.

### Creating Logical Volumes

Inside our volume group rion, we’ll now define three logical volumes:
```bash
lvcreate -L 32G rion -n root      # Root filesystem
lvcreate -L 8G rion -n swap       # Swap volume
lvcreate -l 100%FREE rion -n home # All remaining space goes to home
```

This gives us the following logical volume structure:

- `/dev/rion/root` → mounted as `/`
- `/dev/rion/swap` → swap area
- `/dev/rion/home` → user data and home directory (spanning both encrypted disks)

**Note:** Depending on your LVM setup, logical volumes might appear as `/dev/mapper/rion-root`, `/rion-swap`, etc., or as `/dev/rion/root`, depending on naming conventions. Just make sure to use consistent names in your bootloader and `/etc/fstab`.

You can verify the layout with:
```bash
lvs
```

Here is my actual output:
```bash
LV   VG       Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  home rion -wi-ao---- <1.78t
  root rion -wi-ao---- 32.00g
  swap rion -wi-ao----  8.00g
```

## Formatting the Partitions

Now that the logical volumes and boot partition are set up, it’s time to format everything so it can be used by the system. We’ll format the logical volumes inside our cryptlvm volume group — and the boot partition — as follows:

**Format the Root Filesystem:**
```bash
mkfs.ext4 /dev/rion/root
```

**Format the Home Partition**
```bash
mkfs.ext4 /dev/rion/home
```

**Format the Swap Volume**
```bash
mkswap /dev/rion/swap
```

**Format the EFI System Partition (Boot)**
```bash
mkfs.fat -F32 /dev/nvme0n1p1
```

### Mounting All Partitions

With all partitions formatted, it’s time to mount them in preparation for installing Arch Linux.

**Mount the Root Filesystem**
```bash
mount /dev/rion/root /mnt
```

**Create and Mount the Home Directory**
```bash
mkdir /mnt/home
mount /dev/rion/home /mnt/home
```

**Mount the EFI Boot Partition**
```bash
mkdir /mnt/boot
mount /dev/nvme0n1p1 /mnt/boot
```

**Activate Swap**
```bash
swapon /dev/rion/swap
```

Your filesystems are now mounted and ready. The final directory structure at this point looks like this:
```bash
/mnt
├── boot          → /dev/nvme0n1p1
├── home          → /dev/rion/home
└── (root)        → /dev/rion/root
```

### Installing the Base Arch Linux System

Now that all partitions are mounted, it's time to install a minimal, clean base system onto your new encrypted setup.

**Install Essential Packages**

We’ll install:

`base` → core Arch system

`linux` → the latest kernel

`linux-firmware` → drivers for modern hardware

`systemd-boot` → simple UEFI bootloader

`systemd-networkd` → basic networking

`lvm2` → manage logical volumes

`cryptsetup` → handle LUKS encryption
```bash

pacstrap -K /mnt base linux linux-firmware systemd-boot systemd-networkd systemd-resolved lvm2 cryptsetup
```

`-K` copies your current keyring into the new system to avoid signature issues during package install.

### Configuring the New Installation

**Fstab**
The fstab file can be used to define how disk partitions, various other block devices, or remote filesystems should be mounted into the filesystem.

First we generate an fstab file (use -U or -L to define by UUID or labels, respectively):
`genfstab -U /mnt >> /mnt/etc/fstab`

With the base system installed, it’s time to chroot into it and finalize the configuration.

**Chroot into Your New System**
This changes your root directory to the newly installed system, allowing you to configure it as if you had booted into it directly.
```bash

arch-chroot /mnt
```

Your prompt should now change to `root@archiso /]#` — you're inside the new system.

**fstab — Filesystem Table**
The fstab file defines how devices and partitions are mounted automatically during boot. We already generated it in the previous step, but it’s a good idea to inspect it:
```bash

cat /etc/fstab
```

Here’s an example of what it might look like with your setup:

```bash
# <file system>                        <mount point>  <type>  <options>                        <dump> <pass>

# Encrypted root volume
UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx  /      ext4    rw,relatime                         0      1

# EFI system partition
UUID=52CE-47A9                            /boot   vfat    rw,relatime,fmask=0022,dmask=0022,codepage=437,iocharset=iso8859-1,shortname=mixed,utf8,errors=remount-ro  0 2

# Encrypted home volume
UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx  /home   ext4    rw,relatime                         0      2

# Swap is typically not listed in fstab but enabled via mkswap/swapon
```

*Replace the placeholder UUIDs with actual values from blkid if you ever need to regenerate this file manually.*

**Locale**

Locales are used by glibc and other locale-aware programs or libraries for rendering text, correctly displaying regional monetary values, time and date formats, alphabetic idiosyncrasies, and other locale-specific standards.

I personally uncomment only en_US.UTF-8 UTF-8 in the /etc/locale.gen file. However, you may uncomment additional locales if needed — for example, someone living in Bulgaria might also enable bg_BG.UTF-8 UTF-8.

Once we are done, we need to generate them by running:
```bash
locale-gen
```

As a last step in this section, let’s execute the following in order to create the locale.conf(5) file and set the LANG variable accordingly:
```bash
echo LANG=en_US.UTF-8 > /etc/locale.conf
export LANG=en_US.UTF-8
```

**Timezone Setup**

Select your timezone (optional but helpful interactive method):
```bash
tzselect
```

*This gives you a preview but doesn’t configure the system.*

Link your chosen timezone to /etc/localtime. In my case is Sofia, Bulgaria:

```bash
ln -sf /usr/share/zoneinfo/Europe/Sofia /etc/localtime
```

Set the hardware clock to UTC:
```bash

hwclock --systohc --utc
```

*This ensures your system clock is in sync with the hardware clock and consistent across reboots.*

**Virtual Console (Keyboard & Font Configuration)**

Configure the keyboard layout and (optionally) the font for your TTY environment:
```bash
cat <<EOF > /etc/vconsole.conf
KEYMAP=us
FONT=latarcyrheb-sun32
EOF
```

*You can omit the FONT line or change KEYMAP=us to another layout if you prefer a different keymap.*

**Set Hostname**

Assign a name to your machine. In my case, it is:
```bash
cat <<EOF > /etc/hostname
ryan
EOF
```

**Configure /etc/hosts**
```bash
cat <<EOF > /etc/hosts
127.0.0.1	localhost
::1		    localhost
127.0.1.1	ryan.localdomain ryan
EOF
```

*Replace ryan with your actual hostname if you chose a different one above.*

**Mkinitcpio**

mkinitcpio is a utility used to create the initial RAM filesystem (initramfs) — a small, temporary root file system loaded into memory at boot time. Its purpose is to:

- Mount the real root filesystem
- Load essential kernel modules
- Handle early boot logic (e.g., unlocking encrypted volumes, assembling LVM)

Without a proper initramfs, your system wouldn’t know how to boot into the actual Linux environment — especially if you're using advanced setups like LUKS encryption, LVM, or RAID. Now run:

```bash
mkinitcpio -P
```

The command output should look like this:

```bash
==> Building image from preset: /etc/mkinitcpio.d/linux.preset: 'default'
  -> -k /boot/vmlinuz-linux -c /etc/mkinitcpio.conf -g /boot/initramfs-linux.img
==> Starting build: 6.14.4-arch1-2
  -> Running build hook: [base]
  -> Running build hook: [systemd]
  -> Running build hook: [autodetect]
  -> Running build hook: [keyboard]
  -> Running build hook: [sd-vconsole]
  -> Running build hook: [block]
==> WARNING: Possibly missing firmware for module: wd719x
==> WARNING: Possibly missing firmware for module: aic94xx
  -> Running build hook: [sd-encrypt]
  -> Running build hook: [sd-lvm2]
  -> Running build hook: [filesystems]
  -> Running build hook: [fsck]
==> Generating module dependencies
==> Creating gzip-compressed initcpio image: /boot/initramfs-linux.img
==> Creating gzip-compressed initcpio image: /boot/initramfs-linux-fallback.img
==> Image generation successful
```

*Don’t worry about those two warnings, the XPS 13 doesn’t have any hardware on board that needs those drivers.*

**Microcode**

Modern processors occasionally require microcode updates to fix hardware-level bugs and improve system stability. These updates are especially important on Linux, as they can prevent subtle issues like random crashes, freezes, or erratic behavior — particularly under load.

Since my system is equipped with an AMD processor, I’ll be installing the AMD microcode package. If you use an Intel CPU instead, you should install `intel-ucode` instead.

To install the AMD microcode package:
```bash
pacman -Sy amd-ucode
```

This package will automatically provide the necessary firmware to the kernel at boot time. The AMD microcode will be included in your initramfs or passed via your bootloader (depending on the setup in the next steps).

## Setting Up the Bootloader

`systemd-boot` is a simple, fast, and minimal UEFI boot manager that comes bundled with systemd. Unlike GRUB, it doesn't require complex configuration or scripts — instead, you define boot entries as plain text files.

It works perfectly for UEFI systems and integrates seamlessly with setups involving LUKS, LVM, and microcode.

**Install systemd-boot to the EFI Partition**

Since your /boot partition is mounted at /mnt/boot, and you're already chrooted into /mnt, run:
```bash
bootctl install
```

This installs systemd-boot to your EFI system partition and creates /boot/loader.

**Configure the Loader**

Create the loader configuration file:
```bash
cat <<EOF > /boot/loader/loader.conf
default arch
timeout 3
editor 0
EOF
```
- default arch → sets the default boot entry (we'll create arch.conf next)
- timeout 3 → wait 3 seconds before auto-booting
- editor 0 → disables on-boot edit prompt (you can set this to 1 for debugging)

**Create the Boot Entry**

The boot entry tells systemd-boot exactly how to start your Arch Linux system: which kernel to load, which initramfs to use, and most importantly — how to unlock your encrypted volumes and find your root filesystem.

Since this setup uses two encrypted disks — one for the root volume (/) and another for the extended LVM that includes /home — we must explicitly specify both UUIDs in the boot entry. This is critical: if you omit one, the system won’t know how to unlock both encrypted containers and boot will fail.

**How to Find Your UUIDs**

You can list all available partition UUIDs using:
```bash

blkid
```

Look for lines like these (example output):
```bash
/dev/nvme0n1p2: UUID="11111111-aaaa-bbbb-cccc-111111111111" TYPE="crypto_LUKS"
/dev/nvme1n1p1: UUID="22222222-dddd-eeee-ffff-222222222222" TYPE="crypto_LUKS"
```

These UUIDs will be used in your rd.luks.name kernel parameters.

This is what your /boot/loader/entries/arch.conf should look like when using two LUKS-encrypted disks — one for the root volume and another for /home, both inside the same LVM group:
```bash

mkdir -p /boot/loader/entries
cat <<EOF > /boot/loader/entries/arch.conf
title   Arch Linux
linux   /vmlinuz-linux
initrd  /amd-ucode.img
initrd  /initramfs-linux.img
options rd.luks.name=11111111-aaaa-bbbb-cccc-111111111111=rion rd.luks.name=22222222-dddd-eeee-ffff-222222222222=cryptlvm root=/dev/rion/root rw
EOF
```

**Explanation of Key Options:**

```bash
rd.luks.name=UUID=NAME
→ Unlocks the LUKS container with the given UUID and maps it to /dev/mapper/NAME

In my case:

rion → first disk (/dev/nvme0n1p2) containing the root volume
cryptlvm → second disk (/dev/nvme1n1p1) contributing to the LVM volume group
root=/dev/rion/root → Specifies the root filesystem inside the unlocked volume
rw → Mounts the root filesystem as read-write
```

Now is time to set a root password:
```bash
passwd
```

You’ll be prompted to enter and confirm the password.

Install sudo:
```bash
pacman -Sy sudo
```

sudo lets non-root users execute administrative commands securely.

Before configuring sudo, I prefer to use vim as my text editor. Since I haven’t installed it yet, I’ll do that now:
```bash

pacman -Sy vim
```

Then set it as the default editor for the session:
```bash

export EDITOR=vim
```
Configure sudo Permissions via visudo
Now safely edit the sudoers file:
```bash

visudo
```
This will open `/etc/sudoers` in vim.
*In Vim:*
Press `/` and type wheel → this searches for the correct line
Use `n` to jump to:
```bash

#%wheel ALL=(ALL:ALL) ALL
```

Press `i` to enter insert mode, then remove the `#`. Press Esc, type `:wq`, and press Enter to save and exit.
This line:
`%wheel ALL=(ALL:ALL) ALL`
allows users in the `wheel` group to run any command with `sudo`, which is exactly what you want for administrative tasks.
*visudo also checks for syntax errors before saving, which prevents breaking sudo access.*

## Enabling Networking with systemd

In this guide, we’re using `systemd-networkd` for managing network interfaces and `systemd-resolved` for DNS resolution — a minimal and native systemd-based networking setup. We already installed both components during the pacstrap step. So now, all we need to do is enable and configure the services:

```bash
systemctl enable systemd-networkd
systemctl enable systemd-resolved
```

Then link the DNS resolver file:
```bash
ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
```

Create a Basic DHCP Configuration for Ethernet
```bash
cat <<EOF > /etc/systemd/network/20-wired.network
[Match]
Name=enp*

[Network]
DHCP=yes
EOF
```

*Adjust* Name=enp* to match your actual interface. You can find it by running:
```bash

ip link
```

**Exit the Chroot and Reboot**

Your system is now fully configured. The final step is to leave the chroot environment, unmount all filesystems, and reboot into your new encrypted Arch Linux installation.

Now type `exit` and unmount all mounted partitions. 
Use `-R` to recursively unmount everything under `/mnt`:
```bash

umount -R /mnt
```
and then reboot
```bash
reboot
```
Make sure to remove the USB stick or ISO from which you booted the installer, so the system starts from the disk.

Once the system boots, unlocks the encrypted disks, and logs you into your shell:

Log in as your user.
Then check for IP address with `ip a`.
You should see your interface (e.g. enp1s0) with an IP assigned.
Test internet connection:
```bash

ping -c 3 archlinux.org
```

If you see replies, your network setup is working perfectly.

Congratulations — you now have a clean, secure, fully-encrypted Arch Linux installation using LUKS, LVM, and systemd-boot.

**Conclusion**

From here on, it’s entirely up to you how to shape your system — whether to keep it minimal or install a full desktop environment.

Until recently, I used dwm as my only window manager — fast, tiling, and highly configurable. But I decided to move away from X11, the legacy display server, and switch to Wayland, a modern protocol that offers better performance, security, and smooth rendering.

So I chose to try something new — and installed the COSMIC desktop, which for me has been nothing short of amazing.
