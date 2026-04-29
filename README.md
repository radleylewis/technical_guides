📍Chiang Mai, Thailand

#### YouTube Tutorial 
See the video tutorial [here](https://youtu.be/fFxWuYui2LI).

###### 10 May 2025

# Arch Install [btrfs + encryption + zram + timeshift + cosmic + secure boot]

Good morning, good afternoon or good evening, wherever you are reading this from. These installation instructions form the foundation of the Arch system that I use on my own machine. While it's important to always consult the official Arch wiki install guide [here](https://wiki.archlinux.org/title/Installation_guide), sometimes you may find your preferences deviating from the official guide, and so my intention here is to provide a walkthrough on setting up your own system with the following:
   - [btrfs](https://btrfs.readthedocs.io/en/latest/): A feature rich, copy-on-write filesystem for Linux.  
   - [encryption](https://gitlab.com/cryptsetup/cryptsetup/): LUKS disk encryption based on the dm-crypt kernel module.  
   - [zram-generator](https://github.com/systemd/zram-generator): RAM compression for memory savings.   
   - [timeshift](https://github.com/linuxmint/timeshift): A system restore tool for Linux.  
   - [COSMIC Alpha](https://system76.com/cosmic/): A New Desktop Environment.  

My intention is to keep this guide up-to-date, and any feedback is more than welcome. 

Let's get started!  

## Step 1: Creating a bootable Arch media device

Here we will follow the Arch wiki:
a. Acquire an installation image [here](https://archlinux.org/download/).
b. Verify the signature on the downloaded Arch ISO image (1.2 of the installation guide).
c. Write your ISO to a USB (check out [this](https://www.scaler.com/topics/burn-linux-iso-to-usb/) guide)
d. Insert the USB into the device you intend to install Arch linux on and boot into the USB.

## Step 2: Setting Up Our System with the Arch ISO

a. [optional] if you would like to ssh into your target machine you will need to:
- Create a password for the ISO root user with the `passwd` command; and,
- Ensure that `ssh` is running with `systemctl status sshd` (if it isn't start it with `systemctl start sshd`).
b. Set the console keyboard layout (US by default):
- list available keymaps with `localectl list-keymaps`; and,
- load the keymap with `loadkeys <your keymap here>`.
c. [optional] set the font size with `setfont ter-132b`.
d. Verify the UEFI boot mode `cat /sys/firmware/efi/fw_platform_size`. This installation is written for a system with a 64-bit x64 UEFI. This isn't required, but if you are on a different boot mode, consult section 1.6 of the official guide.
e. Connect to the internet:
- I use the `iwctl` utility for this purpose; and,
- Confirm that your connection is active with `ping -c 2 archlinux.org`.
f. [optional] Obtain your IP Address with `ip addr show`, and now you're ready to ssh into your target machine.
g. Set the timezone:
- `timedatectl list-timezones`;
- `timedatectl set-timezone Asia/Bangkok` (replace Asia/Bangkok with your preferred timezone); and,
- `timedatectl set-ntp true`.
h. Partition your disk:
- list your partitions with `lsblk`;
- delete the existing partitions on the target disk with `gdisk` or `fdisk` [WARNING: your data will be lost]
- create two partitions:
> !NOTE: The official Arch Linux installation guide suggests implementing a swap partition and you are welcome to take this route. You could also create a swap subvolume within BTRFS, however, snapshots will be disabled where a volume has an active swapfile. In my case, I have opted instead for `zram` which works by compressing data in RAM, thereby stretching your RAM further. zram is only active when your RAM is at or close to capacity.  

- **efi** = 1GB    
- **main** = allocate all remaining space (or as otherwise fit for your specific case) noting that BTRFS doesn't require pre-defined partition sizes, but rather allocates dynamically through subvolumes which act in a similar fashion to partitions but don't require the physical division of the target disk.
    
i. format your main partition:  
- setup encryption: `cryptsetup luksFormat /dev/nvme0n1p2`  
- open your encrypted partition: `cryptsetup luksOpen /dev/nvme0n1p2 main`
- format your partition: `mkfs.btrfs /dev/mapper/main`  
- mount your main partition for installation: `mount /dev/mapper/main /mnt`  
- now we need to `cd` into the `/mnt` directory with `cd /mnt`  
- create our subvolumes:  
  **root**: `btrfs subvolume create @`  
  **home**: `btrfs subvolume create @home`
  > !NOTE: you are welcome to create your own subvolume names, but make sure you know what you are doing, because these subvolumes will also be referenced later when taking snapshots with timeshift.  
- go back to the original (/home/root) directory with `cd -`
- unmount our mnt partition: `umount /mnt`
- create home mounting point `mkdir /mnt/home`
- mount our subvolumes: `mount -o noatime,ssd,compress=zstd,discard=async,subvol=@ /dev/mapper/main /mnt`
- mount our subvolumes: `mount -o noatime,ssd,compress=zstd,discard=async,subvol=@home /dev/mapper/main /mnt/home`
j. format your efi partition:
- efi: `mkfs.fat -F32 /dev/nvme0n1p1`
- create boot mounting point `mkdir /mnt/boot`  
- mount our efi partition with `mount /dev/nvme0n1p1 /mnt/boot`  
k. install base packages: `pacstrap -K /mnt base linux linux-firmware`
l. generate the file system table: `genfstab -U -p /mnt >> /mnt/etc/fstab` (you can check this with `cat /mnt/etc/fstab`)
m. change root into the new system: `arch-chroot /mnt`

You are now working from within in your new arch system - i.e. not from the ISO - and you will now see that your prompt start with `#`. Great work so far!

## Step 3: Working Within Our New System

We are now working within our Arch system on our device, but it's important to note that we can't yet reboot our machine. Let's continue with a few steps that we need to repeat again (such as setting our root password, timezones, keymaps and language) given the previous settings were in the context of our ISO.

a. set your local time and locale on your system: 
- `ln -sf /usr/share/zoneinfo/Asia/Bangkok /etc/localtime` (this is in your system, not on the iso)
- `hwclock --systohc`
- in `vim /etc/locale.gen` uncomment your locale, write and exit and then run `locale-gen`
- `echo "LANG=en_US.UTF-8" >> /etc/locale.conf` for locale
- `echo "KEYMAP=us" >> /etc/vconsole.conf` for keyboard (select yours as appropriate)

b. change the hostname `echo "t480" >> /etc/hostname` (feel free to customise to your case, the t480 in my case is for the Lenovo t480 I am installing Arch on).

c. set your root password: `passwd`

d. set up a new user (replace `rad` with your preferred username):
- create `useradd -m -g users -G wheel rad`; 
- give your user a password with `passwd rad` (you will be prompted to enter a password);  
- create a sudoers directory with `mkdir /etc/sudoers.d`;  
- set the sudoers directory permissions `chmod 755 /etc/sudoers.d`;  
- add your user to the sudoers group: `echo "rad ALL=(ALL) ALL" >> /etc/sudoers.d/rad`; and,   
- set the permissions for your sudoers user file `chmod 0440 /etc/sudoers.d/rad`.

e. set mirrorlist `pacman -S reflector && reflector -c Thailand -a 12 --sort rate --save /etc/pacman.d/mirrorlist` (once again you can substitute Thailand with the location relevant to you)  

Next, we will install all of the packages we need for our system. It's imperative to always know what you are doing, and what you are installing!

> !NOTE: you could of course install all of the following packages together, but I have broken them up so they are easier to reason about.

f. install the main packages that our system will use:
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

g. install the following based on the manufacturer of your CPU:
  - **intel:** `pacman -S intel-ucode`
  - **amd**: `pacman -S amd-ucode`

h. install your Desktop Environment and Display Manager of choice:
```bash
pacman -S cosmic ly
```
> !NOTE: I am using COSMIC by System76, but you can just as easily install gnome, kde or whichever tiling window manager or graphical user environment that you would like at this stage. 

i. install other useful packages:
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

j. edit the mkinitcpio file for encrypt:
**NOTE**: this guide is `udev-based` so your HOOKS should look something like:
```
HOOKS=(base udev ... encrypt filesystems ...)
```
`systemd-based` implementations will use `sd-encrypt` instead of `encrypt`.
- `vim /etc/mkinitcpio.conf` and search for HOOKS;
- add encrypt (before filesystems hook);
- OPTIONAL: add `usbhid` & `atkbd` to the MODULES (enables external keyboard at LUKS prompt);
- remove `fsck` from HOOKS (this is a no-op for btrfs which uses `btrfsck` externally);  
- add `btrfs` to the MODULES; and,
- recreate with `mkinitcpio -P`

k. setup grub for the bootloader so that the system can boot linux:
- `grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB`
> NOTE: `--disable-shim-lock` disables GRUB's built in shim lock protocol support. This flag is optional, and should only be passed if you are using custom keys (not the Microsoft Signed shim mechanism).
- run blkid and obtain the UUID for the main partition: `blkid`
- edit the grub config `vim /etc/default/grub`
- `GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet zswap.enabled=0 cryptdevice=UUID=<your-uuid-from-blkid>:main root=/dev/mapper/main"`
- make the grub config with `grub-mkconfig -o /boot/grub/grub.cfg`

l. enable services:
- network manager with `systemctl enable NetworkManager`
- bluetooth with `systemctl enable bluetooth`
- ssh with `systemctl enable sshd`
- ly display manager with `systemctl enable ly.service`
- firewall with `systemctl enable firewalld`
- reflector `systemctl enable reflector.timer`
- `systemctl enable fstrim.timer`
- `systemctl enable acpid`

Now for the moment of truth. Make sure you have followed these steps above carefully, then reboot your system with the `reboot` command.

## Step 4: Tweaking our new Arch system

When you boot up you will be presented with the grub bootloader menu, and then, once you have selected to boot into arch linux (or the timer has timed out and selected your default option) you will be prompted to enter your encryption password. Upon successful decryption, you will be presented with the ly greeter. Enter the password for the user you created earlier. 

a. install [paru](https://github.com/Morganamilo/paru):
```bash
sudo pacman -S --needed base-devel
git clone https://aur.archlinux.org/paru.git
cd paru
makepkg -si
```

b. install `zram-generator`:
**NOTE:** `zram` and `zswap` are both good options. `zram` compresses data and stores them in RAM resulting in less SSD wear. `zswap` uses a compressed in RAM cache that sits in front of a disk based swap partition. This guide uses `zram` but if you regularly exceed your RAM ceiling, `zswap` is preferable.
```bash
sudo pacman -Syu zram-generator
```
create file at: /etc/systemd/zram-generator.conf and add the below contents [guide here](https://forum.manjaro.org/t/howto-install-and-configure-zram-using-zram-generator/168610?page=2)  
```bash
[zram0]
zram-size = ram / 2
compression-algorithm = zstd
swap-priority = 100
fs-type = swap
```
Then reboot. 

c. install [timeshift](https://github.com/linuxmint/timeshift):
```bash
paru -S timeshift timeshift-autosnap
sudo timeshift --list-devices
sudo timeshift --create --comments "[10MAY2025] start of time" --tags D
sudo systemctl edit --full grub-btrfsd
```
> !NOTE:  
> rm : ExecStart=/usr/bin/grub-btrfsd --syslog /.snapshots  
> add: ExecStart=/usr/bin/grub-btrfsd --syslog -t  
```bash
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

## Next: Optional Further Steps
1. install `power-profiles-daemon`:
> !NOTE: you may also like to check out `tlp`, although for my use case `tlp` does not work with COSMIC's power applet (see next step).
```bash
sudo pacman -Syu power-profiles-daemon
sudo systemctl enable --now power-profiles-daemon.service
```
2. install [auto-cpufreq](https://github.com/AdnanHodzic/auto-cpufreq):
```bash
paru -S auto-cpufreq  
sudo systemctl enable --now auto-cpufreq.service  
```
3. install `brave-browser`:
```bash
paru -S brave-browser
```

## Security
The following steps are the foundation of a secure Arch system. There are additional steps that you may of course take to harden your system further, and so you should consider this as a starting point. 

### Secure Boot [OPTIONAL]
**Step 1:** Boot into your system BIOS, and:
a. Clear Vendor Keys (if you are not dual booting);
b. Disable Secure Boot; and,
c. Ensure that Setup Mode is enabled.  
**Step 2:** Configure your system for Secure Boot
a. Install `sbctl`
```
sudo pacman -Syu sbctl
```
b. Generate Keys
```
sudo sbctl create-keys
```
c. Enroll your keys
```
sudo sbctl enroll-keys
```
> NOTE: this is the compressed kernel that your bootloader will load at boot. If you are using other kernel versions (e.g. linux-zen, linux-lts) you will need to sign these too!  
d. create the grub image:
- Reinstall grub: 
```sudo grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB --disable-shim-lock --modules="tpm" --recheck```  
> NOTE: `--disable-shim-lock` disables GRUB's built in shim lock protocol support. This flag is optional, and should only be passed if you are using custom keys (not the Microsoft Signed shim mechanism).
- make the grub config: 
```sudo grub-mkconfig -o /boot/grub/grub.cfg```
e. sign the necessary binaries loaded at boot:
```
sudo sbctl sign -s /boot/vmlinuz-linux
sudo sbctl sign -s /boot/EFI/GRUB/grubx64.efi
```
**Step 3:** REBOOT AND ENABLE SECURE BOOT TO ENSURE IT IS WORKING   
a. create a GRUB password:
```
grub-mkpasswd-pbkdf2
vim /etc/grub.d/40_custom
# add the below
set superusers="admin"
password_pbkdf2 admin <generated-hash>
```
b. Restrict access to grub files
```
sudo chmod 600 /boot/grub/grub.cfg
sudo chmod -R 700 /etc/grub.d
```
c. update `/etc/default/grub`:
- set `GRUB_DISABLE_RECOVERY="true"`; and,  
- set `GRUB_DISABLE_OS_PROBER=true`.
d. Make the grub config:
```
grub-mkconfig -o /boot/grub/grub.cfg
```
e. Reboot the machine and enable Secure Boot in the BIOS

That's all folks!
