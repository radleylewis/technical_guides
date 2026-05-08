# Libreboot: Encrypted /boot Guide

This guide covers single password boot on a Librebooted machine. On a Librebooted system, GRUB is baked into the firmware chip rather than installed to disk. This means GRUB itself unlocks your LUKS container at boot, with the initramfs using an embedded keyfile for the second unlock automatically (resulting in a single password prompt).

> [!resources] **Supporting links:**
> - [Libreboot documentation](https://libreboot.org/docs/linux/)
> - [Artix Installation Guide](/lab/guides/artix_runit_install_guide/)

This guide assumes:

- You have Libreboot installed and running successfully.
- You have completed the Artix Installation Guide (or have any GNU/Linux distribution) with `--type luks2 --pbkdf argon2id` when formatting your LUKS partition.

---

### Step 1: Boot with `iomem=relaxed`

The Linux kernel blocks direct memory access by default. You need to pass `iomem=relaxed` as a kernel parameter to allow `flashrom` to access the firmware chip.

At the GRUB menu press `e`, navigate to the `linux` line and append `iomem=relaxed`, then press `Ctrl+x` to boot.

Verify it worked:

```bash
cat /proc/cmdline
```

You should see `iomem=relaxed` in the output.

---

### Step 2: Install Tools

```bash
sudo pacman -S flashrom
wget https://mrchromebox.tech/files/util/cbfstool.tar.gz && tar -zxf cbfstool.tar.gz
```

---

### Step 3: Backup the Current ROM

Take three reads and verify they match before proceeding:

```bash
sudo flashrom -p internal -r backup1.rom
sudo flashrom -p internal -r backup2.rom
sudo flashrom -p internal -r backup3.rom
diff backup1.rom backup2.rom
diff backup2.rom backup3.rom
```

> [!warning] Store these backups somewhere safe. If anything goes wrong you can restore from them using an external programmer.

---

### Step 4: Set Up a Keyfile

Without a keyfile you will be prompted for your LUKS password twice on every boot: once by Libreboot's GRUB to load the kernel, and once by the initramfs to mount root. A keyfile embedded in the initramfs allows the second unlock to happen automatically.

**i.** Generate and register the keyfile:

```bash
# generate keyfile
sudo dd bs=512 count=4 if=/dev/urandom of=/grub_crypto_key.bin

# add it as a LUKS key
sudo cryptsetup luksAddKey /dev/nvme0n1p2 /grub_crypto_key.bin
```

> [!note] If your LUKS container is already open and `/dev/nvme0n1p2` returns an error, find the underlying device with `lsblk -f` and use that instead.

**ii.** Give it read permissions so mkinitcpio can embed it:

```bash
sudo chmod 400 /grub_crypto_key.bin
```

**iii.** Edit `/etc/mkinitcpio.conf` and add the keyfile to the FILES line:

```bash
FILES=(/grub_crypto_key.bin)
```

**iv.** Rebuild the initramfs:

```bash
sudo mkinitcpio -P
```

**v.** Verify the keyfile is embedded:

```bash
lsinitcpio /boot/initramfs-linux.img | grep grub_crypto
```

---

### Step 5: Create the Firmware `grub.cfg`

**i.** Get your LUKS partition UUID:

```bash
blkid -s UUID -o value /dev/nvme0n1p2
```

**ii.** Create a `grub.cfg` file replacing `<uuid>` with your value. Use the version that matches your setup:

```bash
# SPDX-License-Identifier: GPL-3.0-or-later
# Copyright (C) 2014-2016,2020-2021,2023-2025 Leah Rowe <leah@libreboot.org>
# Copyright (C) 2015 Klemens Nanni <contact@autoboot.org>

set prefix=(memdisk)/boot/grub

insmod at_keyboard
insmod usb_keyboard
insmod regexp

terminal_input --append at_keyboard
terminal_input --append usb_keyboard
terminal_output --append cbmemc

if keystatus --shift; then
	terminal_output --append vga_text
else
	gfxpayload=keep
	terminal_output --append gfxterm

	for dt in cbfsdisk memdisk; do
		for it in png jpg; do
			if [ -f (${dt})/background.${it} ]; then
				insmod ${it}
				background_image (${dt})/background.${it}
			fi
		done
	done
fi

if keystatus --ctrl; then
	serial
	terminal_input --append serial
	terminal_output --append serial
fi

set default="0"
if [ -f (cbfsdisk)/timeout.cfg ]; then
	source (cbfsdisk)/timeout.cfg
else
	set timeout=8
fi

if [ -f (cbfsdisk)/keymap.gkb ]; then
	keymap (cbfsdisk)/keymap.gkb
fi

# --- Uncomment the below for LVM setups (update for your UUID and disk) ---
# menuentry 'Artix Linux' {
#    insmod part_gpt
#    insmod cryptodisk
#    insmod luks2
#    insmod lvm
#    insmod btrfs
#    cryptomount -u <uuid>
#    set root='lvm/vg0-root'
#    linux (lvm/vg0-root)/@/boot/vmlinuz-linux root=/dev/vg0/root cryptdevice=UUID=<uuid>:main rootflags=subvol=@ cryptkey=rootfs:/grub_crypto_key.bin loglevel=3 quiet
#    initrd (lvm/vg0-root)/@/boot/initramfs-linux.img
# }

# --- Uncomment the below for standard setups (update for your UUID and disk) ---
# menuentry 'Artix Linux' {
#     insmod part_gpt
#     insmod cryptodisk
#     insmod luks2
#     insmod btrfs
#     cryptomount -u <uuid>
#     set root='crypto0'
#     linux (crypto0)/@/boot/vmlinuz-linux root=/dev/mapper/main cryptdevice=UUID=<uuid>:main rootflags=subvol=@ cryptkey=rootfs:/grub_crypto_key.bin loglevel=3 quiet
#     initrd (crypto0)/@/boot/initramfs-linux.img
# }

if [ -f (cbfsdisk)/grubtest.cfg ]; then
menuentry 'Load test configuration (grubtest.cfg) inside of CBFS  [t]' --hotkey='t' {
	set root='(cbfsdisk)'
	if [ -f /grubtest.cfg ]; then
		configfile /grubtest.cfg
	fi
}
fi

menuentry 'Poweroff  [p]' --hotkey='p' {
	halt
}

menuentry 'Reboot  [r]' --hotkey='r' {
	reboot
}

if [ -f (cbfsdisk)/img/memtest ]; then
menuentry 'Load MemTest86+  [m]' --hotkey='m' {
	set root='cbfsdisk'
	chainloader /img/memtest
}
fi
```

> [!tip] `set timeout=3` shows the menu for 3 seconds before auto-booting. Set it to `0` to boot straight through with no menu.

---

### Step 6: Update the SPI Chip

Flash the updated ROM:

```bash
cp backup1.rom final.rom
./cbfstool final.rom add -n grub.cfg -f grub.cfg -t raw
sudo flashrom -p internal -w final.rom
```

Keep your backup ROMs somewhere safe in case you ever need to recover with an external programmer.

---

That's all folks!
