---
title: "How to Install Arch Linux with Full Disk Encryption and LVM Using systemd-boot"
date: 2025-05-05
topics: ["arch linux", "security"]
---
This guide describes how to install Arch Linux with full disk encryption, Logical Volume Management (LVM), and the minimalist systemd-boot bootloader. The setup uses two NVMe drives, as this reflects my specific hardware configuration. If you're using only one drive, the process remains mostly the same—just adapt the LUKS and LVM steps accordingly.

Let’s get started.

### Hardware for this Guide

- CPU: AMD Ryzen 9 5900X
- GPU: AMD Radeon RX 6900 XT
- Memory: 32GB
- Two NVMe drives, each 1TB
- UEFI-enabled system (BIOS must support UEFI)

### Preparing the Terrain

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

My USB drive is 16GB and recognized as `/dev/sdb`. Now it’s time to write the ISO image onto it using `dd`. **Caution**: This command will irrevocably erase all data on the drive, so double-check the device path before proceeding.

`sudo dd bs=4M if=/path/to/archlinux.iso of=/dev/sdb status=progress oflag=sync`

Wait for the process to complete — it may take a few minutes. Once it’s done, you’re ready to boot from the USB drive and dive into the real journey.

At this point, we should be in front of a prompt:

```
root@archiso ~ #
```

This is our root prompt, and I’ll be shortening that to `$` in this post.

This is an **optional** step, but if the console font is too small or not readable, we can set it up:

```
setfont latarcyrheb-sun32
```

We need an internet connection, so let’s configure the network. I connected via Ethernet, so everything worked out of the box. If you need Wi-Fi, you can set it up by launching `iwctl` (interaction mode with autocompletion). Here are some useful commands:

```bash
 iwctl
 station list                        # Display your wifi stations
 station <INTERFACE> scan            # Start looking for networks with a station
 station <INTERFACE> get-networks    # Display the networks found by a station
 station <INTERFACE> connect <SSID>  # Connect to a network with a station
```
We also need to update our system clock. Let’s use `timedatectl` to ensure the system clock is accurate:

`timedatectl set-ntp true`

To check the service status, we can use:

```
timedatectl status
```

I tend to use this personal naming convention to identify my hardware with different operating systems, you will see this name around when setting things up in a few places, especially when setting up the volumes in LVM. Swap it out for your own.

### Disk Partitioning

Once that’s done, we can begin building up to the installation. Everything except the `/boot` partition will be encrypted, ensuring a secure and flexible system layout for power users who value control and privacy.

Here is my actual disk layout (run `lsblk` to inspect yours):

**Note:** The `/home` logical volume spans both disks, so its device-mapper entry may appear under both `cryptlvm1` and `cryptlvm2` in `lsblk`.

```
NAME            MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINTS
nvme0n1         259:0    0 931.5G  0 disk
├─nvme0n1p1     259:2    0     1G  0 part  /boot
└─nvme0n1p2     259:3    0 930.5G  0 part
  └─cryptlvm1        253:0    0 930.5G  0 crypt
    ├─cryptlvm1-swap 253:2    0     8G  0 lvm   [SWAP]
    ├─cryptlvm1-root 253:3    0    32G  0 lvm   /
    └─cryptlvm1-home 253:4    0   1.8T  0 lvm   /home
nvme1n1         259:1    0 931.5G  0 disk
└─nvme1n1p1     259:4    0 931.5G  0 part
  └─cryptlvm2    253:1    0 931.5G  0 crypt
    └─cryptlvm1-home 253:4    0   1.8T  0 lvm   /home
```

**Note:** The volume group `cryptlvm1` spans across both encrypted disks. The root (`/`), swap, and part of the home volume reside on the first device (`cryptlvm1`, from `nvme0n1p2`). The second device (`cryptlvm2`, from `nvme1n1p1`) is added entirely to extend the `/home` volume.

This setup results in full disk encryption (FDE), with the sole exception of the `/boot` partition, which remains unencrypted to allow the system to boot.

For partitioning, I used the `gdisk` utility to create and manage the GPT layout:


`gdisk /dev/nvme0n1`

You’ll enter the interactive gdisk prompt. Follow these steps:

Create GPT label (if asked)
If the disk is empty or has no GPT, you may be prompted to create one. Type:

```bash
o
```

Confirm with `y` to create a new empty GPT.

Create EFI system partition (1 GB for /boot)

```bash
n
Partition number: 1
First sector: (press Enter for default)
Last sector: +1G
Hex code or GUID: ef00
w
```

Create encrypted LUKS container (rest of the disk)

```bash
n
Partition number: 2
First sector: (press Enter)
Last sector: (press Enter to use remaining space)
Hex code or GUID: 8e00
w
```

I prefer to partition the second (`nvme1n1`) disk (e.g., for clarity or better tooling support), you can create a single GPT partition that spans the entire disk and set its hex code to `8e00`.

Steps:
```bash

gdisk /dev/nvme1n1
```
Then inside `gdisk`:
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

Now that our partitions are in place, it’s time to secure the system. We’ll encrypt two devices: the second partition of the first disk (`/dev/nvme0n1p2`) and the entirety of the second disk (`/dev/nvme1n1p1`). These will be combined into a single LVM volume group that will host our root, swap, and home volumes — all protected by encryption.

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
cryptsetup open /dev/nvme0n1p2 cryptlvm1
```

**Encrypt `nvme1n1p1`:**

```bash
cryptsetup luksFormat /dev/nvme1n1p1
```

Unlock it:
```bash
cryptsetup open /dev/nvme1n1p1 cryptlvm2
```

### Setting Up LVM

Now we’ll create one logical volume group (`cryptlvm1`) across both unlocked encrypted containers. Inside it, we’ll define three logical volumes: one for root, one for swap, and the rest for home.


```bash
pvcreate /dev/mapper/cryptlvm1 /dev/mapper/cryptlvm2
vgcreate cryptlvm1 /dev/mapper/cryptlvm1 /dev/mapper/cryptlvm2
```

**Note:** The names `cryptlvm1` and `cryptlvm2` used in this guide are just examples. You can choose whatever naming convention you prefer — just make sure to use the same name consistently throughout the setup.

### Creating Logical Volumes

Inside our volume group `cryptlvm1`, we’ll now define three logical volumes:
```bash
lvcreate -L 32G cryptlvm1 -n root      # Root filesystem
lvcreate -L 8G cryptlvm1 -n swap       # Swap volume
lvcreate -l 100%FREE cryptlvm1 -n home # All remaining space goes to home
```

This gives us the following logical volume structure:

- `/dev/cryptlvm1/root` → mounted as `/`
- `/dev/cryptlvm1/swap` → swap area
- `/dev/cryptlvm1/home` → user data and home directory (spanning both encrypted disks)

**Note:** Depending on your LVM setup, logical volumes might appear as `/dev/mapper/cryptlvm1-root`, `/cryptlvm1-swap`, etc., or as `/dev/cryptlvm1/root`, depending on naming conventions. Just make sure to use consistent names in your bootloader and `/etc/fstab`.

You can verify the layout with:
```bash
lvs
```

Here is my actual output:
```bash
LV   VG       Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  home cryptlvm1 -wi-ao---- <1.78t
  root cryptlvm1 -wi-ao---- 32.00g
  swap cryptlvm1 -wi-ao----  8.00g
```

## Formatting the Partitions

Now that the logical volumes and boot partition are set up, it’s time to format everything so it can be used by the system. We’ll format the logical volumes inside our `cryptlvm1` volume group — and the boot partition — as follows:

**Format the Root Filesystem:**
```bash
mkfs.btrfs /dev/cryptlvm1/root
```

**Format the Home Partition**
```bash
mkfs.btrfs /dev/cryptlvm1/home
```

**Format the Swap Volume**
```bash
mkswap /dev/cryptlvm1/swap
```

**Format the EFI System Partition (Boot)**
```bash
mkfs.fat -F32 /dev/nvme0n1p1
```

### Mounting All Partitions

With all partitions formatted, it’s time to mount them in preparation for installing Arch Linux.

**Mount the Root Filesystem**
```bash
mount /dev/cryptlvm1/root /mnt
```

**Create and Mount the Home Directory**

```bash
mount --mkdir /dev/crypt/home /mnt/home
```

*Mount the EFI Boot Partition**

```bash
mount --mkdir /dev/nvme0n1p1 /mnt/boot
```

**Activate Swap**
```bash
swapon /dev/cryptlvm1/swap
```

Your filesystems are now mounted and ready. The final directory structure at this point looks like this:
```bash
/mnt
├── boot          → /dev/nvme0n1p1
├── home          → /dev/cryptlvm1/home
└── (root)        → /dev/cryptlvm1/root
```

### Installing the Base Arch Linux System

Now that all partitions are mounted, it's time to install a minimal, clean base system onto your new encrypted setup.

**Install Essential Packages**

We’ll install:

`base` → core Arch system

`linux` → the latest kernel

`linux-firmware` → drivers for modern hardware

```bash

pacstrap -K /mnt base linux linux-firmware
```

`-K` copies your current keyring into the new system to avoid signature issues during package install.

### Configuring the New Installation

**Fstab**:

The **`fstab`** file defines how disk partitions, other block devices, or remote filesystems should be mounted.

First, generate an `fstab` file:`genfstab -U /mnt >> /mnt/etc/fstab`

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

I personally uncomment only **en_US.UTF-8 UTF-8** in the **`/etc/locale.gen`** file. However, you may uncomment additional locales if needed — for example, someone living in Bulgaria might also enable **bg_BG.UTF-8 UTF-8**.

```bash
vim /etc/locale.conf
```

Add the following content (if not already there):

```
LANG="en_US.UTF-8"
```

Next, open **`/etc/locale.gen`** with **vim** to uncomment (remove the `#`) the required locales. For example, if you want to use English (US), uncomment these lines:

```
vim /etc/locale.gen
```

Find and uncomment (remove the `#`):

```
en_US.UTF-8 UTF-8
```

Once we are done, we need to generate them by running:

```bash
locale-gen
```

**Timezone Setup**

Link your chosen timezone to **`/etc/localtime`**. In my case, it’s Sofia, Bulgaria:

```bash
ln -sf /usr/share/zoneinfo/Europe/Sofia /etc/localtime
```

Set the hardware clock to UTC:
```bash

hwclock --systohc
```

*This ensures your system clock is in sync with the hardware clock and consistent across reboots.*

**Install Essential Packages**

Before we proceed, let’s install a few essential packages using **pacman**:

```
pacman -Syu sudo which man-db man-pages texinfo amd-ucode vim
```

Here’s a brief explanation of each package:

- **sudo**: A tool that allows a permitted user to execute a command as the superuser or another user, as specified by the security policy.
- **which**: A utility that shows the full path of shell commands. It’s useful to find where programs are installed.
- **man-db**: A database used by the **man** command to display manual pages for various commands and programs.
- **man-pages**: A collection of manual pages for Unix/Linux commands and system calls, providing help and documentation directly in the terminal.
- **texinfo**: A documentation system that enables you to create formatted info files, used for software documentation.
- **amd-ucode**: Microcode updates for AMD processors, which can help improve system stability and fix hardware-level bugs.
- **vim**: A highly configurable text editor, used for editing system configuration files (and coding) in a terminal.

**Configure the keyboard layout and (optionally) the    font for your TTY environment**:

```
vim /etc/vconsole.conf
```

Add the following content to the file:

```
KEYMAP=us
FONT=latarcyrheb-sun32
```

*You can omit the `FONT` line or change `KEYMAP=us` to another layout if you prefer a different keymap.*

------

### **Set Hostname**

To assign a name to your machine, open the `/etc/hostname` file in **vim**:

```
vim /etc/hostname
```

Enter your hostname, for example:

```
ryan
```

------

### **Configure `/etc/hosts`**

Open the `/etc/hosts` file using **vim**:

```
vim /etc/hosts
```

Add the following content:

```
127.0.0.1    localhost
::1          localhost
127.0.1.1    ryan.localdomain ryan
```

*Replace "ryan" with your actual hostname if you chose a different one above.*

### **Network Setup**

We will use `systemd-networkd` for managing network interfaces and `systemd-resolved` for DNS resolution, providing a minimal and native systemd-based networking setup. These components were already installed during the **pacstrap** step, so we can now enable and configure the services.

**Enable the necessary services**:

```
systemctl enable systemd-networkd.service
systemctl enable systemd-resolved.service
```

**Edit the network configuration file**:

```
vim /etc/systemd/network/20-wired.network
```

Add the following content:

```
[Match]
Name=enp7s0

[Network]
DHCP=yes
```

*Make sure to replace `enp7s0` with the actual name of your network interface. You can find it by running:*

```
ip link
```

**Link the DNS resolver file**:

```
ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
```

This will set up DHCP for the network interface and ensure proper DNS resolution with `systemd-resolved`.

**Mkinitcpio**

mkinitcpio is a utility used to create the initial RAM filesystem (initramfs) — a small, temporary root file system loaded into memory at boot time. Its purpose is to:

- Mount the real root filesystem
- Load essential kernel modules
- Handle early boot logic (e.g., unlocking encrypted volumes, assembling LVM)

First, make sure the lvm2 package is installed:
```bash
pacman -Syu lvm2
```
Open the configuration file:
```bash
vim /etc/mkinitcpio.conf
```


Locate the `HOOKS` line and make sure it looks like this:
```bash
HOOKS=(base systemd autodetect modconf kms keyboard sd-vconsole block sd-encrypt lvm2 filesystems fsck)
```

Configure the virtual console font in **`/etc/vconsole.conf`**:

```
vim /etc/vconsole.conf
```

Add the following:

```
FONT=latarcyrheb-sun32
```

Finally, regenerate the initramfs:

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
  -> Running build hook: [lvm2]
  -> Running build hook: [filesystems]
  -> Running build hook: [fsck]
==> Generating module dependencies
==> Creating gzip-compressed initcpio image: /boot/initramfs-linux.img
==> Creating gzip-compressed initcpio image: /boot/initramfs-linux-fallback.img
==> Image generation successful
```

*Don’t worry about those two warnings, the XPS 13 doesn’t have any hardware on board that needs those drivers.*

### **Bootloader Installation**

**Exit chroot** and install the bootloader:

```
exit
bootctl --esp-path=/mnt/boot install
```

If you encounter an error (due to a potential bug), follow the steps below to resolve it:

**Fixing the bug (if any error occurs)**:

Reformat the EFI partition and reinstall the bootloader:

```
mkfs.fat -F32 /dev/nvme0n1p1
mount /dev/nvme0n1p1 /mnt/boot
bootctl --esp-path=/mnt/boot install
```

**Chroot into the new system** and install the bootloader again:

```
arch-chroot /mnt
bootctl install
```

### **Configure the Loader**

**Create the loader configuration file** using **vim**:

```
vim /boot/loader/loader.conf
```

Add the following content:

```
default arch
timeout 3
editor 0
```

- `default arch`: Sets **Arch Linux** as the default boot entry.
- `timeout 3`: Waits for 3 seconds before automatically booting.
- `editor 0`: Disables the on-boot edit prompt (set to `1` if you want to enable it for debugging purposes).

------

### **Create the Boot Entry**

The boot entry tells **systemd-boot** how to start your Arch Linux system. It defines:

- Which kernel to load
- Which initramfs to use
- How to unlock encrypted volumes and mount the root filesystem

Since this setup uses **two encrypted disks** (one for the root volume `/` and another for the extended LVM that includes `/home`), you need to explicitly specify both UUIDs in the boot entry. It is **critical** to specify both UUIDs; if you omit one, the system will not know how to unlock both encrypted containers, and booting will fail.

------

### **How to Find Your UUIDs**

First, use the `blkid` command to list the UUIDs of your partitions:

```
blkid
```

Example output (you will have something similar):

```
/dev/nvme0n1p2: UUID="11111111-aaaa-bbbb-cccc-111111111111" TYPE="crypto_LUKS"
/dev/nvme1n1p1: UUID="22222222-dddd-eeee-ffff-222222222222" TYPE="crypto_LUKS"
```

**Copy the UUIDs** into the boot entry using **vim**:

Open the **arch.conf** file in **vim**:

```
vim /boot/loader/entries/arch.conf
```

**Use vim to insert the UUIDs** into the `options` line:

- Press `:` to enter command mode in vim.
- Type `.!blkid` and press **Enter**. This command will output the UUIDs of your partitions directly into the vim buffer.

```
:.!blkid
```

This will append the result of `blkid` directly into the vim editor. You will see output like:

```
/dev/nvme0n1p2: UUID="11111111-aaaa-bbbb-cccc-111111111111" TYPE="crypto_LUKS"
/dev/nvme1n1p1: UUID="22222222-dddd-eeee-ffff-222222222222" TYPE="crypto_LUKS"
```

**Copy the relevant UUIDs**:

- Now, simply copy the **UUIDs** from the `blkid` output (in this case, `11111111-aaaa-bbbb-cccc-111111111111` and `22222222-dddd-eeee-ffff-222222222222`).

**Edit the `options` line** to specify the correct UUIDs for your LUKS-encrypted partitions.

Update the `options` line to look like this, replacing the UUIDs you copied:

```
options rd.luks.name=11111111-aaaa-bbbb-cccc-111111111111=cryptlvm1 rd.luks.name=22222222-dddd-eeee-ffff-222222222222=cryptlvm2 root=/dev/cryptlvm1/root rw
```

**Remove the `blkid` output** (optional):

After copying the UUIDs, you can delete the unnecessary `blkid` output in vim. Simply press `d` to delete lines in normal mode or use `dd` to delete the whole line.

------

### **Create the Boot Entry File**

Add the following content:

```
title   Arch Linux
linux   /vmlinuz-linux
initrd  /amd-ucode.img
initrd  /initramfs-linux.img
options rd.luks.name=11111111-aaaa-bbbb-cccc-111111111111=cryptlvm1 rd.luks.name=22222222-dddd-eeee-ffff-222222222222=cryptlvm2 root=/dev/cryptlvm1/root rw
```

------

### **Explanation of Key Options**

- **`rd.luks.name=UUID=NAME`**: This option unlocks the LUKS container using the specified UUID and maps it to `/dev/mapper/NAME`.
  - `cryptlvm1` → First disk (`/dev/nvme0n1p2`) containing the root volume.
  - `cryptlvm2` → Second disk (`/dev/nvme1n1p1`) contributing to the LVM volume group.
- **`root=/dev/cryptlvm1/root`**: Specifies the root filesystem inside the unlocked volume.
- **`rw`**: Mounts the root filesystem as **read-write**.

Now is time to set a root password:
```bash
passwd
```

You’ll be prompted to enter and confirm the password.

### **Create a New User and Set Password**

Create a new user and add them to the **wheel** group:

```
useradd -G wheel -m simeon
passwd simeon
```

This will create a user **simeon** and allow administrative access via **sudo**.

------

### **Set the Default Editor**

Set **vim** as the default editor for the session:

```
export EDITOR=vim
```

------

### **Configure sudo Permissions via visudo**

To allow the user **simeon** to execute administrative commands, safely edit the sudoers file with **visudo**:

```
visudo
```

This will open the **`/etc/sudoers`** file in **vim**.

In **vim**:

1. Press `/` and type `wheel` to search for the line.

2. Use `n` to jump to:

   ```
   #%wheel ALL=(ALL:ALL) ALL
   ```

3. Press `i` to enter insert mode, then **remove the `#`** at the beginning of the line.

4. Press **Esc**, type `:wq`, and press **Enter** to save and exit.

This line:

```
%wheel ALL=(ALL:ALL) ALL
```

Allows users in the **`wheel`** group (including **simeon**) to run any command with **sudo**, granting administrative privileges.

*Note: `visudo` checks for syntax errors, preventing issues that might break **sudo** access.*

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

Then check for IP address with `ip a`. You should see your interface (e.g. `enp1s0`) with an IP assigned.

Test internet connection:

```bash

ping -c 3 archlinux.org
```

If you see replies, your network setup is working perfectly.

Congratulations — you now have a clean, secure, fully-encrypted Arch Linux installation using LUKS, LVM, and systemd-boot.

**Conclusion**

At this point, your system is fully set up and ready for customization. Whether you prefer a minimal setup or want to install a full desktop environment, the choice is yours.

I currently use a **customized Hyprland** environment on [**Omarchy**](https://omarchy.org/), which provides a fast, efficient, and highly tailored workspace. The beauty of Arch Linux is its flexibility, allowing you to create an environment that fits your exact needs, whether you opt for lightweight setups or a more feature-rich desktop experience.

Now that your system is configured, you have a solid base to explore and experiment with various tools, applications, and configurations. The possibilities are endless!
