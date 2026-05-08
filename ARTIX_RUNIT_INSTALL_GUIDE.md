# Artix Installation Guide

Good morning, good afternoon or good evening, wherever you are reading this from. These installation instructions form the foundation of the Artix system that I use on my own Librebooted T480.

The Artix Linux documentation can be found [here](https://wiki.artixlinux.org/Main/Installation).

This guide uses the following stack:

- [runit](http://smarden.org/runit/): A fast, dependable init system and service supervisor (Artix is systemd-free).
- [btrfs](https://btrfs.readthedocs.io/en/latest/): A feature-rich, copy-on-write filesystem for Linux.
- [LUKS](https://gitlab.com/cryptsetup/cryptsetup/): Full disk encryption based on the dm-crypt kernel module.
- [LVM](https://sourceware.org/lvm2/): Logical Volume Manager, providing stable volume names across reinstalls.
- [zram](https://www.kernel.org/doc/html/latest/admin-guide/blockdev/zram.html): Compressed in-memory block device used as swap, built into the Linux kernel.
- [Qtile](https://qtile.org/): A full-featured, hackable tiling window manager built/configured with Python.
- [ly](https://github.com/fairyglade/ly): A lightweight TUI display manager.

> [!note] **BIOS vs UEFI**
> This guide includes instructions for both BIOS/SeaBIOS and UEFI systems. Steps that differ between the two are clearly marked with **[BIOS]** and **[UEFI]** labels.

> [!note] **btrfs subvolumes**
> This guide creates the following subvolumes:
>
> | Subvolume    | Mount point   | Purpose                                  |
> | ------------ | ------------- | ---------------------------------------- |
> | `@`          | `/`           | root                                     |
> | `@home`      | `/home`       | user files survive a root rollback       |
> | `@snapshots` | `/.snapshots` | required by snapper as its own subvolume |
> | `@log`       | `/var/log`    | logs persist across rollbacks            |
> | `@cache`     | `/var/cache`  | package cache, excluded from snapshots   |

> [!note] **zram**
> This guide uses zram instead of a swap partition. zram creates a compressed in-memory block device that acts as swap, requiring no disk partition and no additional packages beyond `zram-generator`.

> [!note] **LVM and reinstalls**
> LVM logical volumes are referenced by name (e.g. `/dev/vg0/root`) rather than UUID. This means your firmware GRUB config does not need to change on reinstall as long as you recreate the volume group and logical volume with the same names.

> [!note] **runit service packages**
> On Artix, many packages require a separate `-runit` config package to include the runit service directory. The base package alone will not register a service.

Let's get started!

---

## Step 1: Creating a Bootable Artix Media Device

Here we will follow the Artix documentation:

i. Acquire an installation image [here](https://iso.artixlinux.org/isos.php) - select the **runit** variant;

ii. Verify the signature on the downloaded ISO image;

iii. Write your ISO to a USB (check out [this](https://www.scaler.com/topics/burn-linux-iso-to-usb/) guide); and,

iv. Insert the USB into your target device and boot into it. Log in as `root`.

> [!tip] OPTIONAL: Increase the console font size
>
> ```bash
> setfont latarcyrheb-sun32
> ```

---

## Step 2: Setting Up Our System with the Artix ISO

> [!tip] OPTIONAL: SSH into your target machine
>
> - Set a password for the ISO root user with `passwd`;
> - Start the ssh service with `sv start sshd`; and,
> - Obtain your IP address with `ip addr show` (you can now ssh in from another machine).

i. Set the console keyboard layout (US by default):

- list available keymaps with `ls -R /usr/share/kbd/keymaps`; and,
- load your keymap with `loadkeys <your-keymap>`.

ii. Verify your boot mode: the following returns nothing or an error on BIOS, and a list of variables on UEFI:

```bash
ls /sys/firmware/efi/efivars
```

iii. Connect to the internet:

- the Artix ISO ships with `connman` already running (wired connections are automatic); and,
- confirm your connection with `ping -c 2 artixlinux.org`.

iv. Update the system clock:

```bash
sv start ntpd
```

v. Partition your disk:

- list your disks with `lsblk`; and,
- open your target disk with `cfdisk /dev/nvme0n1`.

> [!warning] The following step will delete all data on the target disk. Ensure you have selected the correct disk before proceeding. If the disk already has partitions, delete them all within cfdisk before proceeding. If the disk is busy, run `wipefs -a /dev/nvme0n1` first.

**[BIOS]** Select `dos` label type and create the following partitions:

| Partition        | Size            | Type             |
| ---------------- | --------------- | ---------------- |
| `/dev/nvme0n1p1` | 2MB             | BIOS Boot        |
| `/dev/nvme0n1p2` | Remaining space | Linux filesystem |

**[UEFI]** Select `gpt` label type and create the following partitions:

| Partition        | Size            | Type             |
| ---------------- | --------------- | ---------------- |
| `/dev/nvme0n1p1` | 1GB             | EFI System       |
| `/dev/nvme0n1p2` | Remaining space | Linux filesystem |

Then **Write** and **Quit**.

vi. Format your partitions:

**[UEFI only]** Format the EFI System Partition (ESP):

```bash
mkfs.fat -F32 /dev/nvme0n1p1
fatlabel /dev/nvme0n1p1 ESP
```

Set up LUKS, LVM, and btrfs:

```bash
# set up LUKS encryption
cryptsetup luksFormat --type luks2 --pbkdf argon2id /dev/nvme0n1p2
cryptsetup luksOpen /dev/nvme0n1p2 main

# set up LVM inside the LUKS container
pvcreate /dev/mapper/main
vgcreate vg0 /dev/mapper/main
lvcreate -l 100%FREE vg0 -n root
vgchange -ay

# format as btrfs inside the logical volume
mkfs.btrfs -L MAIN /dev/vg0/root
```

vii. Create btrfs subvolumes:

```bash
mount /dev/vg0/root /mnt
cd /mnt

btrfs subvolume create @
btrfs subvolume create @home
btrfs subvolume create @snapshots
btrfs subvolume create @log
btrfs subvolume create @cache

cd -
umount /mnt
```

viii. Mount subvolumes:

```bash
# create mount points
mkdir -p /mnt/{home,.snapshots,var/log,var/cache}

# root
mount -o noatime,compress=zstd,discard=async,subvol=@ /dev/vg0/root /mnt

# home
mount -o noatime,compress=zstd,discard=async,subvol=@home /dev/vg0/root /mnt/home

# snapshots
mount -o noatime,compress=zstd,discard=async,subvol=@snapshots /dev/vg0/root /mnt/.snapshots

# log
mount -o noatime,compress=zstd,discard=async,subvol=@log /dev/vg0/root /mnt/var/log

# cache
mount -o noatime,compress=zstd,discard=async,subvol=@cache /dev/vg0/root /mnt/var/cache
```

ix. **[UEFI only]** Mount the EFI partition:

```bash
mkdir /mnt/boot
mount /dev/nvme0n1p1 /mnt/boot
```

x. Install the base system:

```bash
basestrap /mnt base base-devel runit elogind-runit lvm2 linux linux-firmware
```

xi. Generate the filesystem table:

```bash
fstabgen -U /mnt >> /mnt/etc/fstab
```

Verify with `cat /mnt/etc/fstab` to check that all subvolumes and boot entries are present and correct before continuing.

xii. Change root into the new system:

```bash
artix-chroot /mnt
```

You are now working within your new Artix base system (i.e. not from the ISO) and you will now see that your prompt starts with `#`. Great work so far!

---

## Step 3: Working Within Our New Base System

We are now working within our Artix system on our device, DO NOT REBOOT yet though! Let's continue with a few steps that we need to repeat again (such as setting our root password, timezones, keymaps and language) given the previous settings were in the context of our ISO.

> [!note] You are now running as root inside the chroot (i.e. no `sudo`).

i. Configure the timezone (replace Asia/Bangkok with your city/timezone):

```bash
ln -sf /usr/share/zoneinfo/Asia/Bangkok /etc/localtime
```

ii. Configure the hardware clock:

```bash
hwclock --systohc
```

iii. Configure locales - in `nvim /etc/locale.gen` uncomment your locale (e.g. `en_US.UTF-8`), write and exit, then run:

```bash
locale-gen
echo "LANG=en_US.UTF-8" >> /etc/locale.conf
echo "KEYMAP=us" >> /etc/vconsole.conf
```

iv. Configure the hostname:

```bash
# T480 will be my hostname, swap in yours
echo "T480" > /etc/hostname
```

Add matching entries to `/etc/hosts`:

```bash
nvim /etc/hosts
127.0.0.1    localhost
::1          localhost
127.0.1.1    T480.localdomain T480
```

v. Set the root password:

```bash
passwd
```

vi. Create a user (replace `rad` with your preferred username):

```bash
useradd -m -G wheel rad
passwd rad
mkdir /etc/sudoers.d
chmod 755 /etc/sudoers.d   # readable, executable by all; writable by root only
echo "rad ALL=(ALL) ALL" >> /etc/sudoers.d/rad
chmod 0440 /etc/sudoers.d/rad   # read-only for root and group; visudo will refuse otherwise
```

vii. Set the mirrorlist (substitute Singapore with your best location/country):

```bash
pacman -S reflector
reflector -c Singapore -a 12 --sort rate --save /etc/pacman.d/mirrorlist
```

viii. Enable the Arch Linux repositories:

Artix does not mirror every Arch package. Adding the Arch repos gives access to packages like `qtile` that are not yet in the Artix repos.

```bash
pacman -S artix-archlinux-support
```

Append the following to `/etc/pacman.conf` after your existing Artix repos (at the bottom):

```
[extra]
Include = /etc/pacman.d/mirrorlist-arch

[multilib]
Include = /etc/pacman.d/mirrorlist-arch
```

Then update:

```bash
pacman -Syu
```

ix. Install core packages:

```bash
pacman -Syu \
  linux-headers \          # kernel build headers
  btrfs-progs \            # btrfs utilities
  lvm2 \                   # LVM tools
  cryptsetup \             # LUKS encryption
  networkmanager \         # network management
  networkmanager-runit \   # runit service file
  network-manager-applet \ # system tray applet
  openssh \                # SSH server/client
  openssh-runit \          # runit service file
  sudo \                   # privilege escalation
  neovim \                 # text editor
  git \                    # version control
  firewalld \              # firewall frontend
  firewalld-runit \        # runit service file
  reflector \              # mirror ranking
  acpid \                  # ACPI event daemon
  acpid-runit \            # runit service file
  cronie \                 # cron scheduler
  cronie-runit \           # runit service file
  grub \                   # bootloader
  elogind \                # login session manager
  dbus \                   # inter-process messaging
  dbus-runit \             # runit service file
  polkit \                 # privilege authorisation
  xdg-user-dirs \          # standard home directories
  zram-generator \         # zram swap
  intel-ucode              # CPU microcode (use amd-ucode for AMD)
```

**[UEFI only]** Also install:

```bash
pacman -S efibootmgr       # EFI boot manager
```

x. Install audio:

```bash
pacman -S \
  pipewire \             # audio server
  pipewire-audio \       # audio support
  pipewire-alsa \        # ALSA routing
  pipewire-pulse \       # PulseAudio replacement
  pipewire-jack \        # JACK replacement
  wireplumber \          # session manager
  alsa-utils \           # ALSA utilities
  sof-firmware           # sound card firmware
```

xi. Install Bluetooth:

```bash
pacman -S \
  bluez \                # Bluetooth stack
  bluez-utils \          # Bluetooth utilities
  bluez-runit            # runit service file
```

xii. Install Qtile, ly, and Wayland dependencies:

```bash
pacman -S \
  ly \                      # display manager
  ly-runit \                # runit service file
  qtile \                   # window manager
  python-pywlroots \        # Wayland backend
  python-pywayland \        # Wayland bindings
  python-xkbcommon \        # keyboard handling
  xorg-xwayland \           # X11 compatibility
  wlr-randr \               # display management
  swaylock \                # screen locker
  swaybg \                  # wallpaper setter
  lxqt-policykit \          # polkit auth agent
  xdg-desktop-portal \      # portal base layer
  xdg-desktop-portal-wlr \  # screen sharing, file pickers
  python-psutil \           # system stats widgets
  qt5-wayland \             # Qt5 Wayland support
  qt6-wayland \             # Qt6 Wayland support
  grim \                    # screenshot tool
  slurp \                   # region selector
  wl-clipboard \            # Wayland clipboard
  xclip \                   # X11 app clipboard
  mako \                    # notification daemon
  brightnessctl \           # brightness control
  playerctl                 # media key support
```

> [!note] If `qtile` is not found, ensure the Arch repos were added correctly in Step 3 (viii) and run `pacman -Syu` before retrying.

> [!note] To verify the correct path for the polkit agent after installation, run `find /usr -name "lxqt-policykit-agent"` and update your Qtile autostart accordingly.

xiii. Install a file manager and supporting utilities:

```bash
pacman -S \
  thunar \               # file manager
  gvfs \                 # trash, USB mounting
  gvfs-mtp \             # Android device support
  pavucontrol            # audio volume control
```

xiv. Install fonts, terminal, window switcher and browser:

```bash
pacman -S \
  man-db \               # manual pages
  man-pages \            # manual content
  texinfo \              # info pages
  ttf-firacode-nerd \    # nerd font
  kitty \                # terminal emulator
  firefox \              # web browser
  rofi                   # window switcher
```

xv. Configure the firewall:

```bash
ln -s /etc/runit/sv/firewalld /etc/runit/runsvdir/default/
sv start firewalld
firewall-cmd --permanent --add-service=ssh
firewall-cmd --reload
```

xvi. Configure mkinitcpio for LUKS + LVM + btrfs:

Edit `/etc/mkinitcpio.conf` and set your MODULES and HOOKS lines to:

> [!tip] OPTIONAL: If you use an external USB keyboard and need it available at the LUKS prompt, add `usbhid` and `atkbd` to MODULES:
>
> ```
> MODULES=(btrfs usbhid atkbd)
> ```

```
MODULES=(btrfs)
HOOKS=(base udev autodetect modconf kms keyboard keymap block lvm2 encrypt filesystems)
```

> [!note] `lvm2` must appear after `block` and before `encrypt` in HOOKS. `fsck` is removed as it is not required for btrfs.

Regenerate the initramfs:

```bash
mkinitcpio -P
```

xvii. Configure GRUB:

**[BIOS]** Install GRUB to your disk:

```bash
grub-install --recheck /dev/nvme0n1
```

**[UEFI]** Install GRUB to the EFI partition:

```bash
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
```

Get the UUID of your LUKS partition and append it to the grub defaults file for reference:

```bash
blkid -s UUID -o value /dev/nvme0n1p2 >> /etc/default/grub
```

Edit `/etc/default/grub` and set the following - replacing `<uuid>` with the value appended at the bottom of the file, then delete that line:

```
GRUB_ENABLE_CRYPTODISK=y
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet cryptdevice=UUID=<uuid>:main root=/dev/vg0/root"
```

> [!note] `GRUB_ENABLE_CRYPTODISK=y` is required because `/boot` resides inside the LUKS-encrypted partition. Without it, GRUB cannot read the kernel and initramfs at boot. This means you will be prompted for your LUKS password **twice** on boot (once by GRUB to load the kernel, and once by the initramfs to mount root).

Generate the GRUB config:

```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

xviii. Enable services:

```bash
ln -s /etc/runit/sv/NetworkManager /etc/runit/runsvdir/default/
ln -s /etc/runit/sv/bluetoothd /etc/runit/runsvdir/default/
ln -s /etc/runit/sv/sshd /etc/runit/runsvdir/default/
ln -s /etc/runit/sv/ly /etc/runit/runsvdir/default/
ln -s /etc/runit/sv/firewalld /etc/runit/runsvdir/default/
ln -s /etc/runit/sv/acpid /etc/runit/runsvdir/default/
ln -s /etc/runit/sv/cronie /etc/runit/runsvdir/default/
ln -s /etc/runit/sv/dbus /etc/runit/runsvdir/default/
```

xix. Set up cron jobs for periodic maintenance:

```bash
# weekly mirrorlist update - substitute Singapore with your location
echo "@weekly root reflector -c Singapore -a 12 --sort rate --save /etc/pacman.d/mirrorlist" >> /etc/cron.d/reflector
```

xx. Reboot:

```bash
exit
umount -R /mnt
swapoff -a
vgchange -an
cryptsetup luksClose main
reboot
```

---

## Step 4: Tweaking Our New Artix System

Upon successful decryption, ly will start. Log in with the user you created earlier and select the Qtile Wayland session.

i. Connect to WiFi if needed:

```bash
nmtui
```

ii. Set up standard home directories for your user:

```bash
xdg-user-dirs-update
```

iii. Install [paru](https://github.com/Morganamilo/paru):

```bash
sudo pacman -S --needed base-devel
git clone https://aur.archlinux.org/paru.git
cd paru
makepkg -si
```

iv. Configure Qtile:

Qtile's config lives at `~/.config/qtile/config.py`. Create the directory if it does not exist:

```bash
mkdir -p ~/.config/qtile
```

On first login Qtile will use its built-in default config if none is present, which is functional but minimal. Check out my config [here](https://github.com/radleylewis/qtile).

---

## Recovery

If you cannot boot into your system, boot from your Artix USB, open the encrypted partition and chroot back in:

```bash
cryptsetup luksOpen /dev/nvme0n1p2 main
vgchange -ay
mount -o noatime,compress=zstd,discard=async,subvol=@ /dev/vg0/root /mnt
mount -o noatime,compress=zstd,discard=async,subvol=@home /dev/vg0/root /mnt/home
artix-chroot /mnt
```

From here you can fix your GRUB config, reinstall packages, or regenerate the initramfs as needed.

---

## Optional Further Steps

i. Install `brave-browser`:

```bash
paru -S brave-browser
```

ii. Add snapper for btrfs snapshots (when ready):

```bash
sudo pacman -S snapper
sudo snapper -c root create-config /
```

> [!note] Snapper expects to manage `/.snapshots` itself. If it complains that `/.snapshots` already exists, run the following to let snapper create and own the subvolume itself:
>
> ```bash
> umount /.snapshots
> btrfs subvolume delete /.snapshots
> snapper -c root create-config /
> ```

Then install the GRUB integration and pacman hook:

```bash
paru -S grub-btrfs snap-pac
```

Enable the pacman hook by ensuring `HookDir` is set in `/etc/pacman.conf`:

```ini
HookDir = /etc/pacman.d/hooks/
```

Then create the hooks directory and hook file:

```bash
sudo mkdir -p /etc/pacman.d/hooks
sudo nvim /etc/pacman.d/hooks/grub-btrfs.hook
```

Paste the following:

```ini
[Trigger]
Operation = Install
Operation = Upgrade
Operation = Remove
Type = Package
Target = *

[Action]
Description = Updating grub-btrfs snapshots...
When = PostTransaction
Exec = /usr/bin/grub-mkconfig -o /boot/grub/grub.cfg
```

Run once manually to populate the GRUB menu immediately:

```bash
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

> [!note] `snap-pac` creates a snapshot automatically before and after every pacman operation (i.e. installs, upgrades and removals). The pacman hook updates the GRUB menu after every transaction, so your snapshots will always be available as a boot option without any manual steps. Arch snapper docs [here](https://wiki.archlinux.org/title/Snapper).

---

That's all folks!
