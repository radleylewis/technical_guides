📍Chiang Mai, Thailand

#### YouTube Tutorial 
See the video tutorial [here](https://www.youtube.com/watch?v=Qgg5oNDylG8).

###### 12 May 2025

# Arch Install [btrfs + encryption + zram + timeshift + cosmic]

Good morning, good afternoon or good evening, whereever you are reading this from. These installation instructions form the foundation of the Arch system that I use on my own machine. While it's important to always consult the official Arch wiki install guide [here](https://wiki.archlinux.org/title/Installation_guide), sometimes you may find your preferences deviating from the the official guide, and so my intention here is to provide a walkthrough on setting up your own system with the following:
   - [btrfs](https://btrfs.readthedocs.io/en/latest/): A feature rich, copy-on-write filesystem for Linux.
   - [encryption](https://gitlab.com/cryptsetup/cryptsetup/): LUKS disk encryption based on the dm-crypt kernel module.
   - [zram](https://www.kernel.org/doc/html/v5.9/admin-guide/blockdev/zram.html): RAM compression for memory savings.
   - [timeshift](https://github.com/linuxmint/timeshift): A system restore tool for Linux.
   - [Cosmic](https://system76.com/cosmic/): A New Desktop Environment

My intention is to keep this guide up-to-date, and any feedback is more than welcome. Let's get started.

## Step 1: Creating a bootable Arch media device

Here we will follow the Arch wiki:
1. Acquire an installation image [here](https://archlinux.org/download/).
2. Verify the signature on the downloaded Arch ISO image (1.2 of the installation guide).
3. Write your ISO to a USB (check out [this](https://www.scaler.com/topics/burn-linux-iso-to-usb/) guide)
4. Insert the USB into the device you intend to install Arch linux on and boot into the USB.

## Step 2: Setting Up Our System with the Arch ISO

1. [optional] if you would like to ssh into your target machine you will need to:
- Create a password for the ISO root user with the `passwd` command; and,
- Ensure that `ssh` is running with `systemctl status sshd` (if it isn't start it with `systemctl start ssdhd`).
2. Set the console keybooard layout (US by default):
- list available keymaps with `localectl list-keymaps`; and,
- load the keymap with `loadkeys <your keymap here>`.
3. [optional] set the font size with `setfont ter-132b`.
4. Verify the UEFI boot mode `cat /sys/firmware/efi/fw_platform_size`. This installation is written for a system with a 62-bit x64 UEFI. This isn't required, but if you are on a different boot mode, consult section 1.6 of the official guide.
5. Connect to the internet:
- I use the `iwctl` utility for this purpose; and,
- Confirm that your connection is active with `ping -c 2 archlinux.org`.
6. [optional] Obtain your IP Address with `ip addr show`, and now you're ready to ssh into your target machine.
7. Set the timezone:
- `timedatectl list-timezones`;
- `timedatectl set-timezone Asia/Bangkok` (replace Asia/Bangkok with your preferred timezone); and,
- `timedatectl set-ntp true`.
8. Partition your disk:
- list your partitions with `lsblk`;
- delete the existing partitions on the target disk [WARNING: your data will be lost]
- create two partitions:
> !NOTE: The official Arch Linux installation guide suggests implementing a swap partition and you are welcome to take this route. You could also create a swap subvolume within BTRFS, however, snapshots will be disabled where a volume has an active swapfile. In my case, I have opted instead of `zram` which works by compressing data in RAM, thereby stretching your RAM further. zram is only active when your RAM is at capacity.  

- **efi** = 300mb    
- **main** = allocate all remaining space (or as otherwise fit for your specific case) noting that BTRFS doesn't require pre-defined partition sizes, but rather allocates dynamically through subvolumes which act in a similar fashion to partitions but don't require the physical division of the target disk.
    
9. format your main partition:  
- setup encryption: `cryptsetup luksformat /dev/nvme0n1p2`  
- open your encrypted partition: `cryptsetup luksOpen /dev/nvme0n1p2 main`
- format your partition: `mkfs.btrfs /dev/mapper/main`  
- mount your main partition for installation: `mount /dev/mapper/main /mnt`  
- now we need into the `/mnt` directory with `cd /mnt`  
- create our subvolumes:  
  **root**: `btrfs subvolume create @`  
  **home**: `btrfs subvolume create @home`
  > !NOTE: you are welcome to create your own subvolume names, but make sure you know what you are doing, because these subvolumes will also be referenced later when taking snapshots with timeshift.  
- go back to the original (root) directory with `cd`
- unmount our mnt partition: `umount /mnt`
- create our boot and home mounting points `mkdir /mnt/{boot,home}`
- mount our subvolumes: `mount -o noatime,ssd,compress=zstd,space_cache=v2,discard=async,subvol=@ /dev/mapper/main /mnt`
- mount our subvolumes: `mount -o noatime,ssd,compress=zstd,space_cache=v2,discard=async,subvol=@home /dev/mapper/main /mnt/home`
10. format your efi partition:
- efi: `mkfs.fat -F32 /dev/nvme0n1p1`
- mount our efi partition with `mount /dev/nvme0np1 /mnt/boot`
11. install base packages: `pacstrap /mnt base`
12. generate the file system table: `genfstab -U -p /mnt >> /mnt/etc/fstab` (you can check this with `cat /mnt/etc/fstab`)
13. change root into the new system: `arch-chroot /mnt`

You are now working from within in your new arch system - i.e. not from the ISO - and you will now see that your prompt start with `#`. Great work so far!

## Step 3: Working Within Our New System

We are now working within our Arch system on our device, but it's important to note that we can't yet reboot our machine. Let's continue with a few steps that we need to repeat again (such as setting our root password, timezones, keymaps and language) given the previous settings were in the context of our ISO.

1. set your local time and locale on your system: 
- `ln -sf /usr/share/zoneinfo/Asia/Bangkok /etc/localtime` (this is in your system, not on the iso)
- `hwclock --systohc`
- locale `nvim /etc/locale.gen` uncomment your locale, write and exit and then run `locale-gen`
- `echo "LANG=en_US.UTF-8" >> /etc/locale.conf` for locale
- `echo "KEYMAP=en..." >> /etc/vconsole.conf` for keyboard
2. change the hostname `echo "x1" >> /etc/hostname` (feel free to customise to your case, the x1 in my case is for the Lenovo x1 Carbon I am installing Arch on).
3. set your root password: `passwd`
4. set up a new user (replace `rad` with your preferred username):
- create `useradd -m -g users -G wheel rad`; 
- give your user a password with `passwd rad` (you will be prompted to enter a password); and,
- add your user to the sudoers group: `echo "rad ALL=(ALL) ALL" >> /etc/sudoers.d/rad`.
5. set mirrorlist `sudo reflector -c Thailand -a 12 --sort rate --save /etc/pacman.d/mirrorlist` (once again you can substitute Thailand with the location relevant to you)  

Next, we will install all of the packages we need for our system. Refer to the bottom of this guide for a short summary on each package being installed. It's imperative to always know what you are doing, and what you are installing!

> !NOTE: you could of course install all of the following packages together, but I have broken them up so they are easier to reason about.

6. install the main packages that our system will use:
```bash
pacman -Syu base-devel linux linux-headers linux-firmware btrfs-progs grub efibootmgr mtools networkmanager network-manager-applet openssh sudo vim git iptables-nft ipset firewalld reflector acpid grub-btrfs
```

7. install the following based on the manufacturer of your CPU:
  - **intel:** `pacman -S intel-ucode`
  - **amd**: `pacman -S amd-code`

8. install your window manager of choice:
```bash
pacman -S qtile xorg lightdm lightdm-gtk-greeter
```
> !NOTE: I am using QTile with X11, but you can just as easily install gnome, kde or whichever tiling window manager or graphical user environment that you would like at this stage. I am using X11 because at the time of writing I have experienced issues with using QTile as a Wayland compositor. I will revisit this from time to time and update this guide accordingly.

9. install other useful packages:
```bash
pacman -S man-db man-pages texinfo bluez bluez-utils pipewire alsa-utils pipewire pipewire-pulse pipewire-jack sof-firmware ttf-firacode-nerd alacritty firefox
```
10. edit the mkinitcpio file for encrypt:
- `vim /etc/mkinitcpio.conf` and search for HOOKS;
- add encrypt (before filesystems hook);
- add `atkbd` to the MODULES (enables external keyboard at device decryption prompt);
- add `btrfs` to the MODULES; and,
- recreate the `mkinitcpio -p linux`
11. setup grub for the bootloader so that the system can boot linux:
- `grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB`
- `grub-mkconfig -o /boot/grub/grub.cfg`
- run blkid and obtain the UUID for the main partitin: `blkid`
- edit the grub config `nvim /etc/default/grub`
- `GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet cryptdevice=UUID=d33844ad-af1b-45c7-9a5c-cf21138744b4:main root=/dev/mapper/main`
- make the grub config with `grub-mkconfig -o /boot/grub/grub.cfg`
12. enable services:
- network manager with `systemctl enable NetworkManager`
- bluetooth with `systemctl enable bluetooth`
- ssh with `systemctl enable sshd`
- lightdm login manager with `systemctl enable lightdm.service`
- firewall with `systemctl enable firewalld`
- reflector `systemctl enable reflector.timer`
- `systemctl enable fstrim.timer`
- `systemctl enable acpid`
- `systemctl enable btrfsd`

Now for the moment of truth. Make sure you have followed these steps above carefully, then reboot your system with the `reboot` command.

## Step 4: Tweaking our new Arch system

When you boot up you will be presented with the grub bootloader menu, and then, once you have selected to boot into arch linux (or the timer has timed out and selected your default option) you will be prompted to enter your encryption password. Upon successful decryption, you will be presented with the lightdm greeter. Enter the password for the user you created earlier. 

QTile out of the box doesn't look great - to say the least -, we still have some work to do. Keep it up!

1. install [paru](https://github.com/Morganamilo/paru):
```bash
sudo pacman -S --needed base-devel
git clone https://aur.archlinux.org/paru.git
cd paru
makepkg -si
```
3. install [zramd](https://github.com/maximumadmin/zramd):
```bash
paru -S zramd
sudo systemctl enable --now zramd.service
```
> !NOTE: you can refer to `lsblk` to see `zram` active (with 8GB by default). To edit your `zram` configuration go to `sudo vim /etc/default/zramd`.
3. install [auto-cpufreq](https://github.com/AdnanHodzic/auto-cpufreq):
```bash
paru -S auto-cpufreq
sudo systemctl enable --now auto-cpufreq.service
```
> !NOTE: you may also like to check out `tlp`, although for my use case `auto-cpufreq` works well enough.
4. install [timeshift](https://github.com/linuxmint/timeshift):
```bash
paru -S timeshift timeshift-autosnap
sudo timeshift --list-devices
sudo timeshift --create --comments "[27JUL2024] start of time" --tags D
sudo systemctl edit --full grub-btrfsd
# NOTE:
# rm : ExecStart=/usr/bin/grub-btrfsd --syslog /.snapshots
# add: ExecStart=/usr/bin/grub-btrfsd --syslog -t
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

## Next: Ricing QTile

Stay tuned for a guide on fully customizing QTile. 
