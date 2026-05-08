# Arch Linux Installation Guide

Good morning, good afternoon or good evening, wherever you are reading this from. These installation instructions form the foundation of the Arch system that I use on my own machine. While it's important to always consult the official Arch wiki, my intention here is to provide a walkthrough on setting up your own system with the following stack:

> [!resources] **Supporting links:**
> - [Arch Linux installation guide](https://wiki.archlinux.org/title/Installation_guide)
> - [YouTube tutorial](https://youtu.be/fFxWuYui2LI)
> - [GitHub (same guide)](https://github.com/radleylewis/arch_installation_guide)

- [btrfs](https://btrfs.readthedocs.io/en/latest/): A feature-rich, copy-on-write filesystem for Linux.
- [encryption](https://gitlab.com/cryptsetup/cryptsetup/): LUKS disk encryption based on the dm-crypt kernel module.
- [zram-generator](https://github.com/systemd/zram-generator): RAM compression for memory savings.
- [timeshift](https://github.com/linuxmint/timeshift): A system restore tool for Linux.
- [COSMIC Alpha](https://system76.com/cosmic/): A new desktop environment by System76.

My intention is to keep this guide up-to-date, and any feedback is more than welcome.

Let's get started!

---

### Step 1: Creating a Bootable Arch Media Device

Here we will follow the Arch wiki:

**i.** Acquire an installation image [here](https://archlinux.org/download/).

**ii.** Verify the signature on the downloaded ISO image (section 1.2 of the installation guide).

**iii.** Write your ISO to a USB (check out [this](https://www.scaler.com/topics/burn-linux-iso-to-usb/) guide).

**iv.** Insert the USB into your target device and boot into it.

---

### Step 2: Setting Up Our System with the Arch ISO

> [!tip] OPTIONAL: SSH into your target machine
>
> - Create a password for the ISO root user with `passwd`; and,
> - Ensure `sshd` is running: `systemctl status sshd` (start with `systemctl start sshd` if not).

**i.** Set the console keyboard layout (US by default):

- list available keymaps with `localectl list-keymaps`; and,
- load your keymap with `loadkeys <your-keymap>`.

> [!tip] OPTIONAL: Set the console font size
>
> ```bash
> setfont ter-132b
> ```

**ii.** Verify the UEFI boot mode:

```bash
cat /sys/firmware/efi/fw_platform_size
```

> [!note] This installation is written for a 64-bit x64 UEFI system. If you are on a different boot mode, consult section 1.6 of the official guide.

**iii.** Connect to the internet:

- use `iwctl` for WiFi; and,
- confirm your connection with `ping -c 2 archlinux.org`.

**iv.** Set the timezone:

```bash
timedatectl list-timezones
timedatectl set-timezone Asia/Bangkok   # replace with your timezone
timedatectl set-ntp true
```

**v.** Partition your disk:

- list partitions with `lsblk`; and,
- open your target disk with `gdisk` or `fdisk` to delete existing partitions and create two new ones.

> [!warning] The following step will delete all data on the target disk. Ensure you have selected the correct disk before proceeding.

> [!note] **zram vs swap partition**
> The official Arch guide suggests a swap partition. This guide uses `zram` instead, which compresses data in RAM and avoids SSD wear. zram is only active when RAM is at or near capacity.

| Partition        | Size            | Type             |
| ---------------- | --------------- | ---------------- |
| `/dev/nvme0n1p1` | 1GB             | EFI System       |
| `/dev/nvme0n1p2` | Remaining space | Linux filesystem |

**vi.** Format your main partition:

```bash
# set up LUKS encryption
cryptsetup luksFormat /dev/nvme0n1p2
cryptsetup luksOpen /dev/nvme0n1p2 main

# format as btrfs
mkfs.btrfs /dev/mapper/main
```

**vii.** Create btrfs subvolumes:

> [!note] Timeshift requires `@` and `@home` as subvolume names. Do not rename them.

```bash
mount /dev/mapper/main /mnt
cd /mnt

btrfs subvolume create @
btrfs subvolume create @home

cd -
umount /mnt
```

**viii.** Mount subvolumes:

```bash
mkdir /mnt/home

mount -o noatime,ssd,compress=zstd,discard=async,subvol=@ /dev/mapper/main /mnt
mount -o noatime,ssd,compress=zstd,discard=async,subvol=@home /dev/mapper/main /mnt/home
```

**ix.** Format and mount the EFI partition:

```bash
mkfs.fat -F32 /dev/nvme0n1p1
mkdir /mnt/boot
mount /dev/nvme0n1p1 /mnt/boot
```

**x.** Install base packages:

```bash
pacstrap -K /mnt base linux linux-firmware
```

**xi.** Generate the filesystem table:

```bash
genfstab -U -p /mnt >> /mnt/etc/fstab
```

- Verify with `cat /mnt/etc/fstab` that all subvolumes and boot entries are present and correct before continuing.

**xii.** Change root into the new system:

```bash
arch-chroot /mnt
```

You are now working within your new Arch system (i.e. not from the ISO) and you will now see that your prompt starts with `#`. Great work so far!

---

### Step 3: Working Within Our New System

We are now working within our Arch system, chrooted in from the ISO. Let's continue with a few steps that we need to repeat (such as setting our root password, timezone, keymaps and language).

> [!warning] Do not reboot yet. Complete all steps in this section before rebooting.

> [!note] You are now running as root inside the chroot (i.e. no `sudo`).

**i.** Configure the timezone and hardware clock:

```bash
ln -sf /usr/share/zoneinfo/Asia/Bangkok /etc/localtime   # replace with your timezone
hwclock --systohc
```

In `nvim /etc/locale.gen`, uncomment your locale (e.g. `en_US.UTF-8`), write and exit, then run:

```bash
locale-gen
echo "LANG=en_US.UTF-8" >> /etc/locale.conf
echo "KEYMAP=us" >> /etc/vconsole.conf
```

**ii.** Configure the hostname:

```bash
echo "t480" > /etc/hostname   # replace with your hostname
```

**iii.** Set the root password:

```bash
passwd
```

**iv.** Create a user (replace `rad` with your preferred username):

```bash
useradd -m -g users -G wheel rad
passwd rad
mkdir /etc/sudoers.d
chmod 755 /etc/sudoers.d    # readable, executable by all; writable by root only
echo "rad ALL=(ALL) ALL" >> /etc/sudoers.d/rad
chmod 0440 /etc/sudoers.d/rad    # read-only for root and group; visudo will refuse otherwise
```

**v.** Set the mirrorlist:

```bash
pacman -S reflector
reflector -c Thailand -a 12 --sort rate --save /etc/pacman.d/mirrorlist
```

> [!note] You could install all of the following packages together, but they are grouped below to make them easier to reason about.

**vi.** Install core packages:

```bash
pacman -Syu \
  base-devel \
  linux-headers \
  btrfs-progs \
  grub \
  efibootmgr \
  mtools \
  networkmanager \
  network-manager-applet \
  openssh \
  sudo \
  neovim \
  git \
  iptables-nft \
  ipset \
  firewalld \
  reflector \
  acpid \
  grub-btrfs
```

**vii.** Install CPU microcode:

- **Intel:** `pacman -S intel-ucode`
- **AMD:** `pacman -S amd-ucode`

**viii.** Install a desktop environment and display manager:

```bash
pacman -S cosmic ly
```

> [!note] This guide uses COSMIC by System76. You can substitute gnome, kde, or any tiling window manager you prefer.

**ix.** Install other useful packages:

```bash
pacman -S \
  man-db \
  man-pages \
  texinfo \
  bluez \
  bluez-utils \
  alsa-utils \
  pipewire \
  pipewire-pulse \
  pipewire-jack \
  sof-firmware \
  ttf-firacode-nerd \
  alacritty \
  firefox
```

**x.** Configure mkinitcpio for LUKS + btrfs:

In `nvim /etc/mkinitcpio.conf`, update MODULES and HOOKS:

```bash
MODULES=(btrfs)
HOOKS=(base udev autodetect modconf kms keyboard keymap block encrypt filesystems)
```

> [!note] This guide is `udev`-based — for a `systemd`-based initramfs replace `encrypt` with `sd-encrypt`. `fsck` is removed as btrfs uses `btrfsck` externally. Add `usbhid` and `atkbd` to MODULES if you need an external keyboard available at the LUKS prompt.

Regenerate the initramfs:

```bash
mkinitcpio -P
```

**xi.** Configure GRUB:

```bash
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
```

Get the UUID of your LUKS partition and append it to the grub defaults file for reference:

```bash
blkid -s UUID -o value /dev/nvme0n1p2 >> /etc/default/grub
```

Edit `/etc/default/grub` and set the following - replacing `<uuid>` with the value appended at the bottom of the file, then delete that line:

```bash
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet zswap.enabled=0 cryptdevice=UUID=<uuid>:main root=/dev/mapper/main"
```

Generate the GRUB config:

```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

**xii.** Enable services:

```bash
systemctl enable NetworkManager    # network management
systemctl enable bluetooth         # Bluetooth
systemctl enable sshd              # SSH server
systemctl enable ly.service        # display manager
systemctl enable firewalld         # firewall
systemctl enable reflector.timer   # mirror ranking
systemctl enable fstrim.timer      # weekly SSD trim
systemctl enable acpid             # ACPI events
```

**xiii.** Reboot:

```bash
reboot
```

---

### Step 4: Tweaking Our New Arch System

When you boot up you will be presented with the GRUB bootloader menu. Once you have selected Arch Linux and entered your encryption password, ly will greet you. Log in with the user you created earlier.

**i.** Install [paru](https://github.com/Morganamilo/paru):

```bash
sudo pacman -S --needed base-devel
git clone https://aur.archlinux.org/paru.git
cd paru
makepkg -si
```

**ii.** Install `zram-generator`:

> [!note] **zram vs zswap**
> `zram` compresses data and stores it in RAM, resulting in less SSD wear. `zswap` uses a compressed in-RAM cache in front of a disk-based swap partition. This guide uses `zram`. If you regularly exceed your RAM ceiling, `zswap` is preferable.

```bash
sudo pacman -Syu zram-generator
```

Create `/etc/systemd/zram-generator.conf`:

```ini
[zram0]
zram-size = ram / 2
compression-algorithm = zstd
swap-priority = 100
fs-type = swap
```

Then reboot.

**iii.** Install [timeshift](https://github.com/linuxmint/timeshift):

```bash
paru -S timeshift timeshift-autosnap
sudo timeshift --list-devices
sudo timeshift --create --comments "initial snapshot" --tags D
sudo systemctl edit --full grub-btrfsd
```

> [!note] In the editor, replace:
> `ExecStart=/usr/bin/grub-btrfsd --syslog /.snapshots`
> with:
> `ExecStart=/usr/bin/grub-btrfsd --syslog -t`

```bash
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

---

### Optional Further Steps

**i.** Install `power-profiles-daemon`:

> [!note] You may also like `tlp`, although for my use case `tlp` does not work with COSMIC's power applet.

```bash
sudo pacman -Syu power-profiles-daemon
sudo systemctl enable --now power-profiles-daemon.service
```

**ii.** Install [auto-cpufreq](https://github.com/AdnanHodzic/auto-cpufreq):

```bash
paru -S auto-cpufreq
sudo systemctl enable --now auto-cpufreq.service
```

**iii.** Install `brave-browser`:

```bash
paru -S brave-browser
```

---

That's all folks!
