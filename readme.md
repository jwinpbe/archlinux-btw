# Arch Linux — Trust Nothing, Snapshot Everything

## Philosophy

This guide is a work in progress. It documents an opinionated Arch Linux installation that attempts to close the gap between a traditional rolling-release Linux desktop and the robustness, security, and user experience paradigms that modern operating systems — particularly macOS and Fedora Silverblue — have been pushing toward.

### The Goal

The aim is not a minimal Arch install. It is a **modern, secure, and resilient desktop** built on the most robust stack currently available on Linux, without sacrificing the transparency and control that makes Arch worth using in the first place.

This means:

- **Full disk encryption** — both `/boot` (LUKS1, unlocked by GRUB) and `/` (LUKS2, unlocked by initramfs). Not just a checkbox, but a meaningful trust chain where the bootloader itself is encrypted.
- **Secure Boot** with custom keys via `sbctl`, so the firmware validates the bootloader before handing off control. This is still being tested and Linux's Secure Boot story is not yet mature — but the infrastructure is here and improving.
- **Btrfs with snapshots** — inspired by macOS Time Machine and openSUSE's snapper integration. Every `pacman` transaction creates a pre/post snapshot. Broken update? Roll back in under a minute.
- **Application sandboxing as a default mindset** — inspired by Silverblue's Flatpak-first approach and macOS's app sandbox model. GUI apps run in Sandman containers, dev tools live in Distrobox environments, and the base system stays lean and reproducible.
- **Wayland-first** — X11 is legacy. This setup is built around Hyprland, Pipewire, and a full Wayland stack. There are still rough edges, especially with NVIDIA, but the ecosystem has matured enough to be daily-driver viable.

### Why Not Just Use Silverblue or macOS?

Silverblue is great. macOS is polished. But:

- macOS locks you to Apple hardware, Apple's release schedule, and Apple's decisions about what your own machine should and shouldn't do. It gets a lot right — disk encryption, sandboxing, firmware security — but you're a guest in their house.
- Fedora Silverblue solves immutability elegantly but you're on GNOME, on RPM, on Red Hat's release schedule, and the moment you need something outside the curated Flatpak ecosystem you're fighting the system instead of using it.
- Both will hold your hand across the street whether you want them to or not.

And yes, FreeBSD would be the obvious answer — a cleaner kernel, a coherent base system, proper jails, ZFS first-class, a ports tree that actually makes sense, an organized ecosystem that isn't a Frankenstein of competing projects, no corporate politics, and a KISS philosophy that is actually taken seriously rather than just cited in a wiki. Unfortunately the FreeBSD community collectively decided desktops are someone else's problem, so here we are, on Linux, like adults who have given up on having nice things. And before you say it — yes, macOS is technically a BSD desktop. If that's what you want, just buy a Mac and save yourself the weekend.

Arch doesn't make many decisions for you — it gives you a base and gets out of the way. If you want more control than that, Gentoo will have you compiling a kernel for three days before you can open a browser. Arch is the sweet spot.

The rolling release model means you're always current, not waiting six months for a kernel that supports your new hardware. And when something breaks, you actually understand enough of the system to fix it instead of filing a support ticket or reinstalling.

Also, if you're being honest, there's something deeply satisfying about booting a machine where you made every decision from partition layout to compositor. That's what this is.

### Honest Limitations

This is still a work in progress, and so is the Linux ecosystem it depends on:

- **Secure Boot** support on Arch is functional but not well-documented in combination with LUKS-encrypted `/boot`. The `sbctl` section in this guide is a starting point, not a verified procedure.
- **NVIDIA on Wayland** has improved dramatically but is not on par with X11 stability. Some compositors and applications still have issues.
- **OSTree / true immutability** is not available on Arch. Snapshots are a good substitute but not equivalent — a bad `pacman -Syu` can still break the running system before you reboot into the snapshot.
- **Sandman** is a young project. Some sandboxing configurations require experimentation.

This guide will evolve as the tooling matures. Contributions and corrections are welcome.

### Application Philosophy

A modern OS should keep the base system clean and reproducible. This setup follows a four-tier approach to installing software, inspired by Silverblue's Flatpak-first model but with more hands-on control:

| Tier | Tool | Use for |
|------|------|---------|
| 1 | pacman / paru | Core system only — drivers, compositor, daemons, libraries |
| 2 | Sandman | GUI apps where you want explicit sandbox control |
| 3 | Flatpak | GUI apps you just want working quickly |
| 4 | Podman / Distrobox | CLI tools, dev runtimes, anything that would pollute the base |

The rule of thumb: if the system wouldn't function without it, it goes in pacman. Everything else lives in a container. The base system should look roughly the same six months from now as it does on install day.

---

## Table of Contents

- [Partition Layout](#partition-layout)
- [1. Partitioning](#1-partitioning)
- [2. Format EFI](#2-format-efi)
- [3. Encrypt /boot with LUKS1](#3-encrypt-boot-with-luks1)
- [4. Encrypt / with LUKS2](#4-encrypt--with-luks2)
- [5. Btrfs — Create Subvolumes](#5-btrfs--create-subvolumes)
- [6. Mount Subvolumes](#6-mount-subvolumes)
- [7. Install Base System](#7-install-base-system)
- [8. Generate fstab](#8-generate-fstab)
- [9. chroot](#9-chroot)
- [10. Basic Configuration](#10-basic-configuration)
- [11. mkinitcpio — Encrypt Hooks](#11-mkinitcpio--encrypt-hooks)
- [12. GRUB Configuration](#12-grub-configuration)
- [13. Keyfile for /boot (crypttab)](#13-keyfile-for-boot-crypttab)
- [14. Enable Services](#14-enable-services)
- [15. Reboot](#15-reboot)
- [Post-Install: GRUB — Windows Dual Boot](#post-install-grub--windows-dual-boot)
- [Post-Install: paru (AUR Helper)](#post-install-paru-aur-helper)
- [Post-Install: Firewall (ufw)](#post-install-firewall-ufw)
- [Post-Install: fail2ban](#post-install-fail2ban)
- [Post-Install: MAC Address Randomization](#post-install-mac-address-randomization)
- [Post-Install: pacman & paru Tuning](#post-install-pacman--paru-tuning)
- [Post-Install: Reflector](#post-install-reflector)
- [Post-Install: Btrfs Scrub](#post-install-btrfs-scrub)
- [Post-Install: earlyoom](#post-install-earlyoom)
- [Post-Install: systemd-timesyncd](#post-install-systemd-timesyncd)
- [Post-Install: fastfetch](#post-install-fastfetch)
- [Post-Install: Snapshot Setup (snapper)](#post-install-snapshot-setup-snapper)
- [Post-Install: zram (Low RAM / High CPU)](#post-install-zram-low-ram--high-cpu)
- [Post-Install: Swapfile Cascade](#post-install-swapfile-cascade)
- [Post-Install: NVIDIA Drivers](#post-install-nvidia-drivers)
- [Post-Install: Secure Boot (sbctl)](#post-install-secure-boot-sbctl)
- [Post-Install: Hyprland Desktop](#post-install-hyprland-desktop)
- [Post-Install: Sandman](#post-install-sandman)
- [Post-Install: KVM / Virt-Manager](#post-install-kvm--virt-manager)
- [Emergency: Chroot from Live CD](#emergency-chroot-from-live-cd)
- [Subvolume Reference](#subvolume-reference)

---

## Partition Layout

| Device | Size | Role |
|--------|------|------|
| `/dev/sda1` | 512M | EFI (FAT32, unencrypted) |
| `/dev/sda2` | 1G | `/boot` — LUKS1 → ext4 |
| `/dev/sda3` | remainder | `/` — LUKS2 → btrfs |

> Adjust device names as needed (`nvme0n1p1`, etc.)

---

## 1. Partitioning

```bash
gdisk /dev/sda
```

```
n → +512M → EF00  (EFI)
n → +1G   → 8300  (boot)
n → (rest) → 8309  (Linux LUKS)
w
```

---

## 2. Format EFI

```bash
mkfs.fat -F32 /dev/sda1
```

---

## 3. Encrypt /boot with LUKS1

GRUB requires LUKS1 to unlock `/boot` at boot time.

```bash
cryptsetup luksFormat --type luks1 /dev/sda2
cryptsetup open /dev/sda2 cryptboot
mkfs.ext4 /dev/mapper/cryptboot
```

---

## 4. Encrypt / with LUKS2

```bash
cryptsetup luksFormat --type luks2 /dev/sda3
cryptsetup open /dev/sda3 cryptroot
```

---

## 5. Btrfs — Create Subvolumes

```bash
mkfs.btrfs /dev/mapper/cryptroot
mount /dev/mapper/cryptroot /mnt

btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@snapshots
btrfs subvolume create /mnt/@var_log
btrfs subvolume create /mnt/@var_pkg
btrfs subvolume create /mnt/@tmp
btrfs subvolume create /mnt/@kvm
btrfs subvolume create /mnt/@swap

umount /mnt
```

---

## 6. Mount Subvolumes

```bash
OPTS="noatime,compress=zstd,space_cache=v2,discard=async"

mount -o subvol=@,$OPTS             /dev/mapper/cryptroot /mnt

mkdir -p /mnt/{home,.snapshots,var/log,var/cache/pacman/pkg,tmp,var/lib/libvirt/images,boot,efi,swap}

mount -o subvol=@home,$OPTS         /dev/mapper/cryptroot /mnt/home
mount -o subvol=@snapshots,$OPTS    /dev/mapper/cryptroot /mnt/.snapshots
mount -o subvol=@var_log,$OPTS      /dev/mapper/cryptroot /mnt/var/log
mount -o subvol=@var_pkg,$OPTS      /dev/mapper/cryptroot /mnt/var/cache/pacman/pkg
mount -o subvol=@tmp,$OPTS          /dev/mapper/cryptroot /mnt/tmp
mount -o subvol=@kvm,$OPTS          /dev/mapper/cryptroot /mnt/var/lib/libvirt/images

# @swap requires noatime and nodatacow — no compression
mount -o subvol=@swap,noatime,nodatacow /dev/mapper/cryptroot /mnt/swap

# Disable Copy-on-Write for VM disk images — must be done while directory is empty
chattr +C /mnt/var/lib/libvirt/images

# Disable CoW and compression for swap — required for swapfile on Btrfs
chattr +C /mnt/swap

mount /dev/mapper/cryptboot /mnt/boot
mount /dev/sda1 /mnt/efi
```

---

## 7. Install Base System

```bash
pacstrap -K /mnt base base-devel linux linux-headers linux-lts linux-lts-headers \
    linux-firmware btrfs-progs grub efibootmgr cryptsetup vim nano opendoas networkmanager zsh \
    amd-ucode
```

> Replace `amd-ucode` with `intel-ucode` if you have an Intel CPU. Do not install both.

---

## 8. Generate fstab

```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

Verify `/mnt/etc/fstab` looks correct before continuing.

---

## 9. chroot

```bash
arch-chroot /mnt
```

---

## 10. Basic Configuration

```bash
# Timezone
ln -sf /usr/share/zoneinfo/<Region>/<City> /etc/localtime
# e.g.: ln -sf /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime
hwclock --systohc

# Locale
echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen
locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf

# Hostname
echo "myhostname" > /etc/hostname
```

### Create user

```bash
useradd -mG wheel,kvm,libvirt,audio,video,storage,optical,network -s /bin/zsh <username>
passwd <username>
```

### Configure doas

```bash
echo "permit persist :wheel" > /etc/doas.conf
chmod 0400 /etc/doas.conf
```

### sudo alias

Many scripts and tools hardcode `sudo`. Add an alias to `~/.zshrc` post-install so they work transparently:

```zsh
alias sudo='doas'
```

### Lock root account

```bash
passwd -l root
```

> Root login is disabled. All privilege escalation goes through `doas`. To recover access, boot from live ISO and `chroot` in.

---

## 11. mkinitcpio — Encrypt Hooks

Find the `HOOKS` line in `/etc/mkinitcpio.conf` and replace it with:

```
HOOKS=(base udev autodetect microcode modconf kms keyboard keymap consolefont block encrypt filesystems fsck)
```

> The key addition is `encrypt` before `filesystems` — this is what unlocks the LUKS2 root at boot.

Rebuild:

```bash
mkinitcpio -P
```

---

## 12. GRUB Configuration

### /etc/default/grub

Get the UUID of `/dev/sda3` (LUKS2 device):

```bash
blkid -s UUID -o value /dev/sda3
```

Set the kernel parameters:

```
GRUB_ENABLE_CRYPTODISK=y

GRUB_CMDLINE_LINUX="cryptdevice=UUID=<UUID-of-sda3>:cryptroot root=/dev/mapper/cryptroot rootflags=subvol=@"
```

> The LUKS1 `/boot` is unlocked by GRUB itself via `GRUB_ENABLE_CRYPTODISK=y`.  
> The LUKS2 `/` is unlocked by the `encrypt` initramfs hook.

### Install GRUB

```bash
grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=GRUB --recheck
grub-mkconfig -o /boot/grub/grub.cfg
```

---

## 13. Keyfile for /boot (crypttab)

GRUB unlocks LUKS1 at boot to read the kernel and initramfs, but `/boot` won't be mounted by the OS unless `cryptboot` is opened again. A keyfile stored on the encrypted root handles this automatically.

### Create the keyfile

```bash
dd bs=512 count=4 if=/dev/random of=/etc/cryptsetup-keys.d/cryptboot.key iflag=fullblock
chmod 600 /etc/cryptsetup-keys.d/cryptboot.key
```

### Add it to the LUKS1 container

```bash
cryptsetup luksAddKey /dev/sda2 /etc/cryptsetup-keys.d/cryptboot.key
```

### /etc/crypttab

```
cryptboot  /dev/sda2  /etc/cryptsetup-keys.d/cryptboot.key  luks
```

> The keyfile lives on the LUKS2-encrypted root, so it's only accessible after `/` is already unlocked. The `encrypt` hook unlocks root first, then `crypttab` is processed and `/boot` is mounted automatically.

---

## 14. Enable Services

```bash
systemctl enable NetworkManager
```

---

## 15. Reboot

```bash
exit
umount -R /mnt
reboot
```

---

## Post-Install: GRUB — Windows Dual Boot

> **Tip:** Rather than relying on GRUB to chainload Windows, a cleaner approach is to manage dual boot directly in your UEFI firmware. Most modern motherboards let you set a boot menu key (commonly F8, F11, or F12) to pick the OS at startup, or have a dedicated boot manager in the UEFI settings where you can reorder or select entries. If your board supports it (common on ASUS, MSI, Gigabyte high-end boards), this is the preferred method — each OS boots natively from its own EFI entry without any chainloading involved.

### Install os-prober

```bash
pacman -S os-prober
```

### Enable os-prober in GRUB

By default GRUB disables os-prober. Edit `/etc/default/grub`:

```
GRUB_DISABLE_OS_PROBER=false
```

### Mount the Windows EFI partition

os-prober needs to see the Windows EFI partition to detect it. If it's not already mounted:

```bash
lsblk  # identify the Windows EFI partition, usually a FAT32 ~100MB partition
mount /dev/sdXY /mnt/windows-efi
```

### Regenerate GRUB config

```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

You should see a line like:

```
Found Windows Boot Manager on /dev/sdXY@/EFI/Microsoft/Boot/bootmgfw.efi
```

Unmount after:

```bash
umount /mnt/windows-efi
```

> If Windows was installed after Arch, it may have overwritten the GRUB EFI entry. Reinstall GRUB with `grub-install` and regenerate the config.

> Windows fast startup (hybrid shutdown) can cause filesystem corruption on shared drives. Disable it in Windows: **Power Options → Turn on fast startup → uncheck**.

---

## Post-Install: Firewall (ufw)

```bash
pacman -S ufw
```

Set default policy and allow SSH if needed:

```bash
ufw default deny incoming
ufw default allow outgoing

# Allow SSH only if you use it — skip otherwise
ufw allow ssh

# Allow KVM/libvirt NAT traffic
ufw allow in on virbr0
ufw allow out on virbr0
```

Enable and start:

```bash
ufw enable
systemctl enable ufw
```

Check status:

```bash
ufw status verbose
```

> The `virbr0` rules prevent ufw from blocking libvirt's virtual network. If you're not using KVM, skip those lines.

---

## Post-Install: fail2ban

Bans IPs that repeatedly fail authentication. Most relevant if SSH is exposed.

```bash
pacman -S fail2ban
```

Create a local jail config at `/etc/fail2ban/jail.local`:

```ini
[DEFAULT]
bantime  = 1h
findtime = 10m
maxretry = 5
backend  = systemd

[sshd]
enabled = true
```

Enable and start:

```bash
systemctl enable --now fail2ban
```

Check banned IPs:

```bash
fail2ban-client status sshd
```

> If you're not running SSH, fail2ban has limited use on a typical desktop. The `[sshd]` jail shown here is the most common starting point — additional jails can be added for other exposed services.

---

## Post-Install: MAC Address Randomization

MAC randomization prevents passive tracking across networks — each time you connect to an unknown network, a different MAC is presented. This is the default behavior on iOS and Android and increasingly expected on a privacy-conscious desktop.

NetworkManager handles this natively with no extra packages.

### Enable global randomization

Create `/etc/NetworkManager/conf.d/mac-randomization.conf`:

```ini
[device]
wifi.scan-rand-mac-address=yes

[connection]
wifi.cloned-mac-address=random
ethernet.cloned-mac-address=random
connection.stable-id=${CONNECTION}/${BOOT}
```

- `wifi.scan-rand-mac-address=yes` — randomizes MAC during WiFi scans, before even associating
- `cloned-mac-address=random` — uses a different random MAC on every connection
- `${CONNECTION}/${BOOT}` — ties the random MAC to the current boot, so it stays stable within a session but changes on reboot

### Use a stable MAC for trusted networks

For your home network, work network, or anywhere MAC filtering or DHCP reservations are in use, you want a consistent MAC. Configure this per-connection in `/etc/NetworkManager/system-connections/<your-network>.nmconnection`:

```ini
[wifi]
ssid=YourNetworkName
cloned-mac-address=permanent

[wifi-security]
# ... your existing auth config
```

Or set it via `nmcli`:

```bash
# Use the real hardware MAC for a specific WiFi network
nmcli connection modify "YourNetworkName" wifi.cloned-mac-address permanent

# Or set a specific fixed MAC
nmcli connection modify "YourNetworkName" wifi.cloned-mac-address "aa:bb:cc:dd:ee:ff"
```

Restart NetworkManager to apply:

```bash
systemctl restart NetworkManager
```

Verify the active MAC:

```bash
ip link show wlan0
```

> `permanent` tells NetworkManager to use the actual hardware MAC burned into the NIC. Use this for trusted networks where you need a consistent identity — DHCP reservations, MAC-filtered networks, or networks where your router assigns a fixed IP to your device.

---

## Post-Install: paru (AUR Helper)

```bash
pacman -S git base-devel
git clone https://aur.archlinux.org/paru.git
cd paru
makepkg -si
```

---

## Post-Install: pacman & paru Tuning

### /etc/pacman.conf

Uncomment or add under the `[options]` block:

```ini
[options]
Color
ILoveCandy
VerbosePkgLists
ParallelDownloads = 5
CheckSpace

# Color         — colored output
# ILoveCandy    — Pac-Man progress bar instead of the default hash bar
# VerbosePkgLists — shows old vs new version table on upgrades
# ParallelDownloads — download multiple packages simultaneously (5 is a sane default)
# CheckSpace    — warns if disk space is insufficient before installing
```

Enable the `multilib` repo for 32-bit support (Steam, Wine, games). Uncomment in `/etc/pacman.conf`:

```ini
[multilib]
Include = /etc/pacman.d/mirrorlist
```

Then sync:

```bash
pacman -Sy
```

### ~/.config/paru/paru.conf

```ini
[options]
BottomUp
CombinedUpgrade

# BottomUp        — reverses search results so best match is closest to the cursor
# CombinedUpgrade — merges repo and AUR upgrades into a single operation
# CleanAfter      — removes build files after install (skipped: prefer manual cache control)
# NewsOnUpgrade   — shows Arch news before upgrades (skipped: personal preference)
# BatchInstall    — queues AUR packages and installs together (skipped: prefer per-package control)
# Devel           — checks -git/-svn packages for upstream changes (skipped: can be slow)
# SudoLoop        — refreshes sudo session during builds (not compatible with doas)
```

---

## Post-Install: Reflector

Automatically updates `/etc/pacman.d/mirrorlist` with the fastest mirrors for your country.

```bash
pacman -S reflector
```

Run once manually:

```bash
reflector --country Brazil --age 12 --protocol https --sort rate --save /etc/pacman.d/mirrorlist
```

> Replace `Brazil` with your country. `--age 12` excludes mirrors not synced in the last 12 hours.

Configure the systemd service at `/etc/xdg/reflector/reflector.conf`:

```
--country Brazil
--age 12
--protocol https
--sort rate
--save /etc/pacman.d/mirrorlist
```

Enable the weekly timer:

```bash
systemctl enable reflector.timer
```

---

## Post-Install: Btrfs Scrub

Btrfs scrub reads all data and metadata, verifying checksums and correcting errors on redundant setups. It should run periodically to catch silent corruption early.

Enable the monthly scrub timer for the root filesystem:

```bash
systemctl enable btrfs-scrub@-.timer
```

To scrub other subvolume mount points (optional):

```bash
systemctl enable "btrfs-scrub@$(systemd-escape -p /home).timer"
```

Run a scrub manually at any time:

```bash
btrfs scrub start /
```

Check scrub status:

```bash
btrfs scrub status /
```

> The `btrfs-scrub@-.timer` unit uses `-` as the escaped form of `/`. systemd handles the mapping automatically.

---

## Post-Install: earlyoom

Kills the most memory-hungry process before the system fully freezes, well before the kernel OOM killer would act. Recommended on low-RAM machines.

```bash
pacman -S earlyoom
systemctl enable --now earlyoom
```

---

## Post-Install: systemd-timesyncd

NTP time sync. Not enabled by default on Arch.

```bash
systemctl enable --now systemd-timesyncd
```

Verify:

```bash
timedatectl status
```

---

## Post-Install: fastfetch

The most important part of any Arch install. Without this, did you even install Arch?

> `neofetch` is unmaintained and removed from the Arch repos. `fastfetch` is the actively maintained successor and significantly faster.

```bash
pacman -S fastfetch
```

Run it:

```bash
fastfetch
```

### Autostart in zsh

Add to `~/.zshrc` so it greets you every time you open a terminal — as nature intended:

```zsh
fastfetch
```

### Configuration

Generate a default config to customize:

```bash
fastfetch --gen-config
```

Config lives at `~/.config/fastfetch/config.jsonc`. You can customize which modules show, their order, colors, logo, and more.

Show a specific logo:

```bash
fastfetch --logo Arch
```

Use a custom image as logo (Kitty terminal required):

```bash
fastfetch --logo /path/to/image.png --logo-type kitty
```

> At this point your system has full disk encryption, Secure Boot, Btrfs snapshots, a sandboxed app stack, a firewall, MAC randomization, and a Wayland compositor. You have earned the right to run fastfetch.

---

## Post-Install: Snapshot Setup (snapper)

```bash
pacman -S snapper snap-pac

# snapper expects /.snapshots to NOT be mounted when creating config
umount /.snapshots
rmdir /.snapshots

snapper -c root create-config /

# Replace the subvol snapper created with the one you already have
btrfs subvolume delete /.snapshots
mkdir /.snapshots
mount -a   # remounts @snapshots from fstab
```

Set permissions so non-root users can browse snapshots (optional):

```bash
chmod 750 /.snapshots
```

---

## Post-Install: zram (Low RAM / High CPU)

Compresses RAM contents on the fly, effectively giving you more usable memory at the cost of CPU cycles. Ideal when you have a capable processor but limited RAM — either by choice, by hardware constraints, or because you checked RAM prices recently and decided you'd rather just suffer.

Install `zram-generator`:

```bash
pacman -S zram-generator
```

Create `/etc/systemd/zram-generator.conf`:

```ini
[zram0]
zram-size = ram * 1.5
compression-algorithm = zstd
writeback-device = none

# Aggressive swap pressure — push to zram early and hard
[zram0.extra]
vm.swappiness = 180
vm.watermark_boost_factor = 0
vm.watermark_scale_factor = 125
vm.page-cluster = 0
```

Enable and start:

```bash
systemctl daemon-reload
systemctl start systemd-zram-setup@zram0.service
```

Verify:

```bash
zramctl
```

> `zstd` gives the best compression ratio at the cost of CPU — ideal for a low-RAM machine with a capable processor. `zram-size = ram * 1.5` means on 8 GB RAM you get a 12 GB compressed swap device. The `swappiness=180` is a kernel hint (valid range 0–200 on modern kernels) that aggressively favors zram over disk swap.

> If you're on a more balanced system with adequate RAM, a more conservative setup makes more sense — lower `zram-size` (0.5x or equal to RAM), `swappiness=100`, and a small swap partition on a fast NVMe as overflow. Combining both gives you compression benefits without thrashing the CPU when memory pressure gets extreme.

---

## Post-Install: Swapfile Cascade

Pairs with zram to create a two-tier swap hierarchy: zram handles the first wave of memory pressure in compressed RAM, the swapfile on NVMe catches the overflow. The kernel uses swap priorities to enforce the order — zram first, disk only when zram is exhausted.

### Create the swapfile

The `@swap` subvolume is already mounted at `/swap` with CoW disabled. Create the swapfile there:

```bash
# Adjust size to your needs — 4G is a reasonable overflow buffer
fallocate -l 4G /swap/swapfile
chmod 600 /swap/swapfile
mkswap /swap/swapfile
swapon --priority 10 /swap/swapfile
```

> zram is assigned priority 100 by default, so the swapfile at priority 10 will always be used last.

### Add to /etc/fstab

```
/swap/swapfile  none  swap  sw,pri=10  0  0
```

### Verify the cascade

```bash
swapon --show
```

You should see both zram0 (high priority) and the swapfile (low priority) listed.

### Priority reference

| Device | Priority | Role |
|--------|----------|------|
| `/dev/zram0` | 100 (default) | First — fast compressed RAM |
| `/swap/swapfile` | 10 | Overflow — NVMe disk |

> On a balanced system with enough RAM, you can skip zram entirely and just use the swapfile with a conservative `swappiness=60`. The cascade only makes sense when RAM is tight enough that you expect zram to actually fill up.

---

## Post-Install: NVIDIA Drivers

Using `nvidia-dkms` covers both `linux` and `linux-lts` with a single package.

```bash
pacman -S nvidia-dkms nvidia-utils nvidia-settings
```

Add `nvidia` modules to `/etc/mkinitcpio.conf`:

```
MODULES=(nvidia nvidia_modeset nvidia_uvm nvidia_drm)
```

Add the kernel parameter to `/etc/default/grub`:

```
GRUB_CMDLINE_LINUX="... nvidia-drm.modeset=1"
```

Rebuild initramfs and GRUB config:

```bash
mkinitcpio -P
grub-mkconfig -o /boot/grub/grub.cfg
```

Prevent the `nvidia-uvm` module from being removed while in use and blacklist `nouveau` — create `/etc/modprobe.d/nvidia.conf`:

```
options nvidia-drm modeset=1
blacklist nouveau
```

> If you later install a kernel update and the DKMS build fails, check `dkms status` and ensure `linux-headers` and `linux-lts-headers` are installed.

---

## Post-Install: Secure Boot (sbctl)

> ⚠️ **This section is a future plan and has not been tested with this setup. Treat it as a reference starting point, not a verified guide. Proceed with caution.**

Secure Boot with custom keys is possible using `sbctl`. The idea is to replace the default OEM keys with your own, sign GRUB and the kernels, and have the firmware reject any unsigned bootloader — giving you a meaningful trust chain on top of the LUKS-encrypted `/boot`.

### Prerequisites

- Secure Boot must be in **Setup Mode** in UEFI (clear existing keys first)
- UEFI must support custom key enrollment

### Install sbctl

```bash
pacman -S sbctl
```

### Check Secure Boot status

```bash
sbctl status
```

Both `Setup Mode` and `Secure Boot` should show as enabled and off respectively before proceeding.

### Create and enroll custom keys

```bash
sbctl create-keys
sbctl enroll-keys -m  # -m includes Microsoft keys, needed for dual boot with Windows
```

### Sign the bootloader and kernels

```bash
sbctl sign -s /efi/EFI/GRUB/grubx64.efi
sbctl sign -s /boot/vmlinuz-linux
sbctl sign -s /boot/vmlinuz-linux-lts
```

The `-s` flag saves the paths so `sbctl` re-signs automatically after kernel or GRUB updates via a pacman hook.

### Verify

```bash
sbctl verify
```

All signed files should show ✓.

### Enable Secure Boot in UEFI

Reboot, enable Secure Boot in firmware settings, and verify:

```bash
sbctl status
```

> Omit `-m` when enrolling keys if you are not dual booting Windows. Enrolling Microsoft keys is required for Windows to boot under Secure Boot.

> If GRUB updates overwrite the signed EFI binary, re-run `sbctl sign` or ensure the pacman hook is active (`sbctl` installs it automatically).

---

## Post-Install: Hyprland Desktop

A complete, functional Wayland desktop — not just Hyprland alone.

### XWayland & X11 Compatibility

Required for running legacy X11 apps under Hyprland:

```bash
pacman -S xorg-xwayland qt5-wayland qt6-wayland xorg-xlsclients
```

- `xorg-xwayland` — runs X11 applications inside the Wayland session
- `qt5-wayland` / `qt6-wayland` — makes Qt apps use native Wayland instead of falling back to XWayland
- `xorg-xlsclients` — lists which apps are currently running under XWayland, useful for debugging

> To check if an app is running natively on Wayland or falling back to XWayland: `xlsclients` will list all X11 clients. If your app appears there, it's using XWayland.

---

### Hyprland + Display Manager

```bash
pacman -S hyprland xdg-desktop-portal-hyprland xdg-desktop-portal-gtk \
    sddm qt6-svg qt6-declarative
```

Enable SDDM:

```bash
systemctl enable sddm
```

---

### Pipewire (Audio)

```bash
pacman -S pipewire pipewire-alsa pipewire-pulse pipewire-jack \
    wireplumber pavucontrol
```

Enable for your user (as the user, not root):

```bash
systemctl --user enable pipewire pipewire-pulse wireplumber
```

---

### Polkit

```bash
pacman -S polkit polkit-gnome
```

Add to your Hyprland config to autostart the polkit agent:

```
exec-once = /usr/lib/polkit-gnome/polkit-gnome-authentication-agent-1
```

---

### Portals & XDG

```bash
pacman -S xdg-utils xdg-user-dirs
xdg-user-dirs-update
```

---

### Fonts

```bash
pacman -S noto-fonts noto-fonts-emoji noto-fonts-extra \
    ttf-jetbrains-mono-nerd ttf-liberation \
    ttf-dejavu ttf-unifont
```

Optional — Chinese (and broader CJK) support:

```bash
pacman -S noto-fonts-cjk
```

> `noto-fonts-cjk` is a large package (~130MB). Only install if you need Chinese, Japanese, or Korean rendering.

---

### Theming (GTK + Qt)

```bash
pacman -S nwg-look qt5ct qt6ct kvantum
```

Set Qt platform theme in `/etc/environment`:

```
QT_QPA_PLATFORM=wayland
QT_QPA_PLATFORMTHEME=qt6ct
GDK_BACKEND=wayland
SDL_VIDEODRIVER=wayland
CLUTTER_BACKEND=wayland
```

---

### Wayland Essentials

```bash
pacman -S wayland wayland-utils wl-clipboard cliphist \
    grim slurp swappy brightnessctl upower playerctl
```

---

### Bar, Launcher, Notifications

```bash
pacman -S waybar wofi mako
```

---

### Screen Lock & Idle

```bash
pacman -S hyprlock hypridle
```

---

### Terminal & File Manager

```bash
pacman -S kitty thunar thunar-archive-plugin thunar-volman \
    gvfs gvfs-mtp file-roller
```

---

### Bluetooth

```bash
pacman -S bluez bluez-utils blueman
systemctl enable bluetooth
```

---

### Keyring

```bash
pacman -S gnome-keyring libsecret seahorse
```

Add to `/etc/pam.d/login` to unlock the keyring on login:

```
auth     optional  pam_gnome_keyring.so
session  optional  pam_gnome_keyring.so auto_start
```

---

### Network Tray

```bash
pacman -S network-manager-applet
```

Autostart in Hyprland config:

```
exec-once = nm-applet --indicator
```

---

### NVIDIA-specific (Hyprland)

Add to `/etc/environment` if using NVIDIA:

```
LIBVA_DRIVER_NAME=nvidia
GBM_BACKEND=nvidia-drm
__GLX_VENDOR_LIBRARY_NAME=nvidia
WLR_NO_HARDWARE_CURSORS=1
```

> `WLR_NO_HARDWARE_CURSORS=1` prevents the invisible cursor bug on NVIDIA under Hyprland.

---

### Full Package Summary

```bash
pacman -S hyprland xdg-desktop-portal-hyprland xdg-desktop-portal-gtk \
    sddm qt6-svg qt6-declarative \
    pipewire pipewire-alsa pipewire-pulse pipewire-jack wireplumber pavucontrol \
    polkit polkit-gnome \
    xdg-utils xdg-user-dirs \
    noto-fonts noto-fonts-emoji noto-fonts-extra ttf-jetbrains-mono-nerd ttf-liberation \
    ttf-dejavu ttf-unifont \
    nwg-look qt5ct qt6ct kvantum \
    wayland wayland-utils wl-clipboard cliphist \
    grim slurp swappy brightnessctl upower playerctl \
    waybar wofi mako \
    hyprlock hypridle \
    kitty thunar thunar-archive-plugin thunar-volman gvfs gvfs-mtp file-roller \
    bluez bluez-utils blueman \
    gnome-keyring libsecret seahorse \
    network-manager-applet \
    xorg-xwayland qt5-wayland qt6-wayland xorg-xlsclients
```

---

## Post-Install: Sandman

Sandman runs GUI and CLI applications inside rootless Podman containers, giving each app its own ephemeral sandbox with explicit control over what's exposed (Wayland, Pipewire, GPU, devices, etc.).

### Dependencies

```bash
pacman -S podman buildah go
```

### Setup subuid/subgid (required for rootless containers)

```bash
echo "<username>:100000:65536" >> /etc/subuid
echo "<username>:100000:65536" >> /etc/subgid
```

### Enable Podman socket (user service)

```bash
systemctl --user enable --now podman.socket
```

### Install Sandman

```bash
git clone https://github.com/julioln/sandman.git
cd sandman
make all
```

> `make all` downloads dependencies, runs tests, compiles, and installs the binary.

### Config location

TOML configs live at `~/.config/sandman/<appname>.toml`. Local persistent storage at `~/.local/share/sandman/<appname>`.

### Example — sandboxed app on Wayland with Pipewire

**`~/.config/sandman/myapp.toml`**

```toml
[Build]
Instructions = '''
FROM archlinux
RUN pacman -Syu myapp --noconfirm
CMD "/usr/bin/myapp"
'''

[Run]
Wayland = true
Pipewire = true
Dri = true
Gpu = true
Fonts = true
Network = "none"
Uidmap = true
Home = false
```

```bash
sandman build myapp
sandman start myapp   # detached
sandman run myapp     # attached
```

> Keep `Network = "none"` unless the app genuinely needs internet access. Enable `Home = true` only when you need data to persist across container runs.

---

## Post-Install: KVM / Virt-Manager

### Install packages

```bash
pacman -S qemu-full virt-manager virt-viewer \
    libvirt dnsmasq iptables-nft \
    edk2-ovmf swtpm \
    bridge-utils dmidecode
```

- `qemu-full` — full QEMU with all architectures and device support
- `edk2-ovmf` — UEFI firmware for VMs (required for EFI boot)
- `swtpm` — software TPM emulator (required for TPM 2.0, e.g. Windows 11)
- `dnsmasq` — required for libvirt's default NAT network

### Enable services

```bash
systemctl enable --now libvirtd
systemctl enable --now virtlogd
```

### Enable default NAT network

```bash
virsh net-autostart default
virsh net-start default
```

### Storage location

VM disk images use the `@kvm` subvolume mounted at `/var/lib/libvirt/images`.

Verify libvirt's default pool points there:

```bash
virsh pool-info default
```

If needed, redefine it:

```bash
virsh pool-destroy default
virsh pool-undefine default
virsh pool-define-as default dir --target /var/lib/libvirt/images
virsh pool-autostart default
virsh pool-start default
```

### UEFI firmware path (OVMF)

When creating a VM in virt-manager, set firmware to UEFI. The OVMF firmware files are at:

```
/usr/share/edk2/x64/OVMF_CODE.fd       # read-only firmware
/usr/share/edk2/x64/OVMF_VARS.fd       # per-VM vars template
```

libvirt reads these automatically from `/etc/libvirt/qemu.conf` — no manual config needed if using virt-manager.

### TPM 2.0

In virt-manager, add a TPM device:
- Model: `TIS`
- Backend: `Emulated`
- Version: `2.0`

`swtpm` handles the emulation automatically. Per-VM TPM state is stored at:

```
/var/lib/libvirt/swtpm/<vm-uuid>/
```

### User access (no password prompt)

Your user was already added to the `kvm` and `libvirt` groups in step 10. Verify:

```bash
groups
```

If missing, add manually:

```bash
doas usermod -aG kvm,libvirt <username>
```

Log out and back in for group changes to take effect.

### libvirt config files reference

| Path | Purpose |
|------|---------|
| `/etc/libvirt/qemu.conf` | QEMU driver config (OVMF paths, security driver, etc.) |
| `/etc/libvirt/libvirtd.conf` | libvirtd daemon config |
| `/var/lib/libvirt/images/` | VM disk images (`@kvm` subvol) |
| `/var/lib/libvirt/swtpm/` | TPM state per VM |
| `/var/log/libvirt/qemu/` | Per-VM logs |
| `/etc/libvirt/qemu/` | Per-VM XML definitions |

---

## Emergency: Chroot from Live CD

Boot from the Arch Linux ISO, then follow these steps to unlock and mount your system for recovery. Useful for: broken GRUB, forgotten doas config, kernel issues, or anything that requires `chroot` access.

### 1. Unlock LUKS containers

```bash
# Unlock /boot (LUKS1)
cryptsetup open /dev/sda2 cryptboot

# Unlock / (LUKS2)
cryptsetup open /dev/sda3 cryptroot
```

### 2. Mount the Btrfs subvolumes

```bash
OPTS="noatime,compress=zstd,space_cache=v2,discard=async"

mount -o subvol=@,$OPTS /dev/mapper/cryptroot /mnt

mkdir -p /mnt/{home,.snapshots,var/log,var/cache/pacman/pkg,tmp,var/lib/libvirt/images,boot,efi}

mount -o subvol=@home,$OPTS      /dev/mapper/cryptroot /mnt/home
mount -o subvol=@snapshots,$OPTS /dev/mapper/cryptroot /mnt/.snapshots
mount -o subvol=@var_log,$OPTS   /dev/mapper/cryptroot /mnt/var/log
mount -o subvol=@var_pkg,$OPTS   /dev/mapper/cryptroot /mnt/var/cache/pacman/pkg
mount -o subvol=@tmp,$OPTS       /dev/mapper/cryptroot /mnt/tmp
mount -o subvol=@kvm,$OPTS       /dev/mapper/cryptroot /mnt/var/lib/libvirt/images

mount /dev/mapper/cryptboot /mnt/boot
mount /dev/sda1 /mnt/efi
```

### 3. chroot

```bash
arch-chroot /mnt
```

You now have a fully functional shell inside your system. From here you can:

- Rebuild GRUB: `grub-install` + `grub-mkconfig`
- Rebuild initramfs: `mkinitcpio -P`
- Fix doas: `echo "permit persist :wheel" > /etc/doas.conf && chmod 0400 /etc/doas.conf`
- Unlock root if locked out: `passwd root`
- Reinstall or fix packages: `pacman -S <pkg>`

### 4. Exit and clean up

```bash
exit
umount -R /mnt
cryptsetup close cryptroot
cryptsetup close cryptboot
reboot
```

> If `arch-chroot` complains about missing tools on the live ISO, install them first: `pacman -Sy arch-install-scripts`.

---

## Subvolume Reference

| Subvolume | Mount Point | Purpose |
|-----------|-------------|---------|
| `@` | `/` | Root filesystem |
| `@home` | `/home` | User home dirs |
| `@snapshots` | `/.snapshots` | snapper snapshots |
| `@var_log` | `/var/log` | Logs (excluded from root snapshots) |
| `@var_pkg` | `/var/cache/pacman/pkg` | Package cache (excluded from snapshots) |
| `@tmp` | `/tmp` | Temp files |
| `@kvm` | `/var/lib/libvirt/images` | KVM disk images |
| `@swap` | `/swap` | Swapfile (CoW disabled, no compression) |