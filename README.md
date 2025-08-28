# Build a Fully Unattended Debian 12 ISO (Preseed)

**Goal:** Produce a Debian 12.11 netinst (amd64) ISO that installs without interaction (no locale/keyboard prompts), using preseed.

**Tested on:** VirtualBox 7.x, Debian 12.11 and 13.0.0

## 1. Prerequisites

Install required packages:

```bash
sudo apt-get update
sudo apt-get install -y xorriso isolinux syslinux-utils rsync whois
```

### Recommended Directory Layout

```
/vm/
├─ iso/
│  ├─ debian-12.11.0-amd64-netinst.iso          # official ISO
│  └─ debian-12.11.0-amd64-netinst-custom.iso   # output
├─ iso/debian-iso/                               # mount point (read-only)
└─ iso/debian-iso-work/                          # working tree (editable)
```

### (Optional) Generate Password Hash

Generate a SHA-512 password hash for the preseed:

```bash
mkpasswd -m sha-512 'YourStrongPassword'
```

## 2. Prepare the Working Tree

Set up variables and prepare the working directory:

```bash
SRC_ISO="/vm/iso/debian-13.0.0-amd64-netinst.iso"
MNT_DIR="/vm/iso/debian-iso"
WRK_DIR="/vm/iso/debian-iso-work"

sudo mkdir -p "$MNT_DIR" "$WRK_DIR"
sudo mount -o loop "$SRC_ISO" "$MNT_DIR"
sudo rsync -a --delete "$MNT_DIR"/ "$WRK_DIR"/
```

## 3. Create preseed.cfg

Create `$WRK_DIR/preseed.cfg` with the following content:

```bash
### =====================================================================
### Debian Preseed - Unattended (Bookworm 12.x)
### =====================================================================

# Early localization (avoid initial dialogs)
d-i debconf/priority string critical
d-i debian-installer/language string en
d-i debian-installer/country string CO
d-i debian-installer/locale string en_US.UTF-8

# Keyboard (US)
d-i keyboard-configuration/ask_detect boolean false
d-i keyboard-configuration/layoutcode string us
d-i keyboard-configuration/modelcode string pc105

# Network
d-i netcfg/choose_interface select auto
d-i netcfg/link_wait_timeout string 3
d-i netcfg/dhcp_timeout string 10
d-i netcfg/get_hostname string no-hostname
d-i netcfg/hostname string no-hostname
d-i netcfg/get_domain string singularit.co
d-i base-installer/install-recommends boolean false

# User
d-i passwd/make-user boolean true
d-i passwd/root-login boolean false
d-i passwd/user-uid string 1000
d-i passwd/username string sysadmin
# Replace with your own SHA-512 hash
d-i passwd/user-password-crypted password $6$mOVKPpiDXNRZd/gH$o460/niXr72jzlr6pHtchtEJYiSQ4OPefm9EAD5NznNZiX.WV/63YRrMubnEJ0g0XWO7qJWxrYIaHxy6KYtN20
d-i passwd/user-fullname string System Administrator
d-i passwd/user-default-groups string audio cdrom video

# Mirror (no conflicts)
d-i mirror/country string manual
d-i mirror/http/hostname string http.us.debian.org
d-i mirror/http/directory string /debian
d-i mirror/http/proxy string

# Popularity contest
d-i popularity-contest/participate boolean false

# Apt setup
d-i apt-setup/contrib boolean true
d-i apt-setup/non-free boolean true
d-i apt-setup/non-free-firmware boolean true
d-i apt-setup/enable-source-repositories boolean false
d-i apt-setup/services-select multiselect security, updates
d-i apt-setup/security_host string security.debian.org
d-i apt-setup/security_path string /debian
d-i apt-setup/security_protocol string https
d-i apt-setup/cdrom/set-first boolean false
d-i apt-setup/disable-cdrom-entries boolean true

# Clock / timezone
d-i clock-setup/utc boolean true
d-i time/zone string America/Bogota
d-i clock-setup/ntp boolean true
d-i clock-setup/ntp-server string us.pool.ntp.org

# Partitioning (guided LVM)
d-i partman-auto/method string lvm
d-i partman-auto-lvm/guided_size string max
d-i partman-lvm/device_remove_lvm boolean true
d-i partman-lvm/confirm boolean true
d-i partman-lvm/confirm_nooverwrite boolean true
d-i partman-md/device_remove_md boolean true
d-i partman-auto/choose_recipe select atomic
d-i partman/confirm_write_new_label boolean true
d-i partman-partitioning/confirm_write_new_label boolean true
d-i partman/choose_partition select finish
d-i partman/confirm boolean true
d-i partman/confirm_nooverwrite boolean true

# Base system
d-i base-installer/kernel/image select linux-image-amd64
d-i base-installer/initramfs-tools/driver-policy select dep

# Packages
d-i pkgsel/upgrade select full-upgrade
tasksel tasksel/first multiselect standard, ssh-server

# GRUB
d-i grub-installer/only_debian boolean true
d-i grub-installer/force-efi-extra-removable boolean true
d-i grub-installer/bootdev string default

# Keep consoles
d-i finish-install/keep-consoles boolean true

# Late command (robust, doesn't fail if files are missing)
d-i preseed/late_command string \
  in-target mkdir -p /home/sysadmin/.ssh ; \
  cp /cdrom/authorized_keys /target/home/sysadmin/.ssh/authorized_keys || true ; \
  chown -R 1000:1000 /target/home/sysadmin/.ssh ; \
  chmod 700 /target/home/sysadmin/.ssh ; \
  chmod 600 /target/home/sysadmin/.ssh/authorized_keys || true ; \
  echo 'sysadmin ALL=(ALL) NOPASSWD:ALL' > /target/etc/sudoers.d/sysadmin ; \
  chmod 0440 /target/etc/sudoers.d/sysadmin ;

# Eject and reboot
d-i cdrom-detect/eject boolean true
d-i finish-install/reboot_in_progress note
```

### Optional: Add SSH Keys

If you want to include SSH authorized keys:

```bash
sudo cp /home/sysadmin/.ssh/authorized_keys .
sudo chmod 644 "$WRK_DIR/authorized_keys" || true
```

## 4. Configure BIOS Boot (ISOLINUX)

⚠️ **Important:** In BIOS, the `append` line must be on ONE line (no line breaks). Breaking the line drops initrd causing kernel panic or ignores parameters.

### Create `$WRK_DIR/isolinux/isolinux.cfg`:

```bash
# D-I config version 2.0
PROMPT 0
DEFAULT install
ONTIMEOUT install
TIMEOUT 10      # 1.0s; use 1 for almost immediate

INCLUDE txt.cfg # loads labels from txt.cfg
```

### Create `$WRK_DIR/isolinux/txt.cfg`:

```bash
default install

label install
    menu label ^Install
    menu default
    kernel /install.amd/vmlinuz
    append auto=true priority=critical file=/cdrom/preseed.cfg preseed/file=/cdrom/preseed.cfg preseed/locale=en_US.UTF-8 locale=en_US.UTF-8 language=en country=CO console-setup/ask_detect=false keyboard-configuration/ask_detect=false keyboard-configuration/xkb-keymap=us keyboard-configuration/layoutcode=us initrd=/install.amd/initrd.gz --- quiet

label installgui
    menu label ^Graphical install
    kernel /install.amd/vmlinuz
    append auto=true priority=critical file=/cdrom/preseed.cfg preseed/file=/cdrom/preseed.cfg preseed/locale=en_US.UTF-8 locale=en_US.UTF-8 language=en country=CO console-setup/ask_detect=false keyboard-configuration/ask_detect=false keyboard-configuration/xkb-keymap=us keyboard-configuration/layoutcode=us initrd=/install.amd/gtk/initrd.gz --- quiet
```

### Verification

In the BIOS menu, select Install and press Tab. You should see:

```
/install.amd/vmlinuz auto=true ... file=/cdrom/preseed.cfg ... initrd=/install.amd/initrd.gz --- quiet
```

## 5. Configure UEFI Boot (GRUB)

Create `$WRK_DIR/boot/grub/grub.cfg`:

```bash
set default="0"
set timeout=0

menuentry 'Graphical install' {
linux /install.amd/vmlinuz auto=true priority=critical file=/cdrom/preseed.cfg preseed/file=/cdrom/preseed.cfg locale=en_US.UTF-8 language=en country=CO keyboard-configuration/ask_detect=false keyboard-configuration/layoutcode=us --- quiet
initrd /install.amd/gtk/initrd.gz
}

menuentry 'Install' {
linux /install.amd/vmlinuz auto=true priority=critical file=/cdrom/preseed.cfg preseed/file=/cdrom/preseed.cfg locale=en_US.UTF-8 language=en country=CO keyboard-configuration/ask_detect=false keyboard-configuration/layoutcode=us --- quiet
initrd /install.amd/initrd.gz
}
```

## 6. (Recommended) Embed preseed.cfg into initrd

This eliminates any race conditions with `/cdrom`:

```bash
cd "$WRK_DIR/install.amd"
mkdir -p initrd-edit && cd initrd-edit
gunzip -c ../initrd.gz | cpio -id
cp "$WRK_DIR/preseed.cfg" ./preseed.cfg
find . | cpio -o -H newc | gzip -9 > ../initrd.gz
cd .. && rm -rf initrd-edit
```

*Note: You may also keep `preseed/file=/preseed.cfg` in kernel cmdline in addition to `file=/cdrom/preseed.cfg`.*

## 7. Build the ISO

Create the custom ISO:

```bash
OUT_ISO="/vm/iso/debian-13.0.0-amd64-netinst-custom.iso"

xorriso -as mkisofs \
  -o "$OUT_ISO" \
  -r -iso-level 3 -V "DEBIAN_CUSTOM" \
  -no-emul-boot -boot-load-size 4 -boot-info-table \
  -b isolinux/isolinux.bin -c isolinux/boot.cat \
  -eltorito-alt-boot -e boot/grub/efi.img -no-emul-boot \
  -isohybrid-gpt-basdat -isohybrid-apm-hfsplus \
  -input-charset utf-8 \
  /vm/iso/debian-iso-work
```

### Cleanup

Unmount the original ISO when done:

```bash
sudo umount "$MNT_DIR" || true
```

## 8. Troubleshooting

### Common Issues and Solutions

| Issue | Cause | Solution |
|-------|-------|----------|
| Language/keyboard screen still appears | Wrong boot firmware (BIOS vs UEFI) or parameters not applied | Check boot firmware type and verify parameters. In BIOS press Tab on Install and inspect the cmdline |
| Kernel panic "unknown-block(0,0)" | Missing `initrd=...` because the append line was split | Keep the append line on ONE line in BIOS configuration |
| `late_command` fails | Files missing under `/cdrom` or sudoers redirection issues | Use the robust block provided and check `/var/log/syslog` |

### Log Files

Check these log files for debugging:
- `/var/log/syslog` - General system logs during installation
- Installation logs in the target system after boot

---

## Summary

This guide creates a fully automated Debian 12 installation ISO that:
- ✅ Skips all interactive prompts
- ✅ Configures US keyboard layout
- ✅ Sets up a sysadmin user with sudo privileges
- ✅ Configures SSH server
- ✅ Works with both BIOS and UEFI systems
- ✅ Uses LVM partitioning
- ✅ Configures Colombian timezone

The resulting ISO will boot and install completely unattended, perfect for automated deployments.
