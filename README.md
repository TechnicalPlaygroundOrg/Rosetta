# Rosetta

A hybrid research operating system that combines functionality from Windows, Mac
and Linux, with the goal of full compatibility with all of them.

> **Vision vs. current state:** The cross-platform compatibility described above is
> the project's research *goal*. The current build is a standard Linux system (see
> below) that runs Linux software. Windows/Mac compatibility is not yet implemented.

---

## State

The current version of Rosetta is a Linux distribution based on **LFS 13.0 (systemd)**
with the following changes and characteristics.

### Kernel
- Linux **6.18.10**, custom-built.
- Boot-critical drivers (**virtio, NVMe, SATA/AHCI, ext4**) are compiled **into** the
  kernel rather than as modules. This means **no initramfs is required**, and the same
  image can boot across different storage backends.
- Common NIC drivers built in — Intel (`e1000`, `e1000e`), Realtek (`r8169`), Broadcom
  (`tg3`), plus `virtio-net` — so networking works on both VMs and physical hardware.

### Networking
- **No static IPs**; uses DHCP on all interfaces to allow easy loading on different VMs
  and machines.
- Interfaces are renamed by **type/driver** (e.g. `ether0`, `wifi0`, `tether0`) via
  systemd `.link` files rather than by MAC address, keeping configuration
  hardware-independent.
- DNS is handled dynamically by **`systemd-resolved`** (no static `resolv.conf`);
  `/etc/hosts` carries no static IP, relying on the `nss-myhostname` NSS module.
- Wi-Fi is provisioned at the IP layer but requires a supplicant
  (`wpa_supplicant` / `iwd`) from BLFS to actually associate — **not yet functional**
  in the base system.

### Identity & Locale
- Hostname: `Rosetta`
- Locale: `C.UTF-8`
- Timezone: `America/Chicago`
- Hardware clock: **UTC**

### Package Installation
- All packages have their own installation scripts.
- An orchestration script runs each one to set up user space.

> *(This subsystem is project-specific and outside the base LFS build; update this
> section to reflect its actual current state — implemented vs. planned.)*

### Bootloader
- Rosetta is set up to run **alongside an existing operating system** using a separate
  GRUB entry (legacy **BIOS** / GPT, with a 1 MB BIOS-boot partition).
- Boot device resolution is by **UUID / PARTUUID**, not device name, so it boots
  regardless of whether the disk enumerates as `vda`, `sda`, or `nvme`.
- It is fully capable of running as the **sole OS** on a system with a few changes
  (running its own `grub-install` so it owns the boot record instead of depending on the
  companion OS's GRUB). See **[Updating the GRUB Entries](#updating-the-grub-entries)**.

---

## Installation / Testing

Due to partitioning and device-naming schemes adopted during development under
Virt-manager, Rosetta currently targets **QEMU/KVM** or a wrapper around it such as
Virt-manager. The ideal companion OS to run alongside Rosetta is currently **Ubuntu**,
but there are no known issues running alongside other operating systems or
distributions. A full compatibility table will be added when the data is available.

> **Firmware:** Rosetta is currently **legacy BIOS only**. The VM must be configured for
> BIOS (SeaBIOS), **not** UEFI. UEFI support requires an EFI System Partition and a UEFI
> GRUB install (planned, not yet implemented).

### Requirements

**Virtual Machine**
- Hypervisor: QEMU/KVM (Virt-manager recommended on Linux)
- Firmware: **BIOS** (not UEFI)
- Disk bus: **virtio** (AHCI/SATA also supported)
- Recommended: 2+ vCPUs, 2 GB+ RAM, ~20 GB disk
- Partition layout (GPT): 1 MB BIOS-boot partition + ext4 root (swap optional)

**Companion OS (for side-by-side mode)**
- Any Linux distribution whose bootloader can chain to or detect Rosetta (Ubuntu
  tested). The companion manages the GRUB menu via an added entry pointing at Rosetta's
  kernel by UUID/PARTUUID.

### Distribution Artifacts

- **Full disk image (`.qcow2`)** — a complete, directly bootable disk. (The development
  image currently contains *both* the companion OS and Rosetta partitions.) Import into a
  new VM as an existing disk and set firmware to BIOS.
- **Filesystem archive (`.tar.xz`)** — Rosetta's root filesystem only. Compact and
  portable, but **not directly bootable**: it must be extracted onto a partitioned disk
  and have GRUB installed before it will boot.

---

### Host Setup

Rosetta runs under QEMU. The underlying host OS (Windows or Linux) only needs to launch
QEMU; all editing of Rosetta happens *inside* the VM via the companion OS (Ubuntu), so
that part of the workflow is identical on every host.

#### On Linux (recommended)

Install QEMU/KVM and, optionally, Virt-manager:

```bash
sudo apt install qemu-system-x86 qemu-utils virt-manager   # Debian/Ubuntu
```

Either manage the VM through the Virt-manager GUI, or launch the disk directly:

```bash
qemu-system-x86_64 \
  -enable-kvm \
  -m 4096 -smp 4 \
  -drive file=/path/to/rosetta.qcow2,if=virtio,format=qcow2 \
  -netdev user,id=n0 -device virtio-net,netdev=n0 \
  -display gtk
```

#### On Windows (via QEMU)

Virt-manager is Linux-only, so on Windows use QEMU directly.

1. Install QEMU for Windows (official builds: <https://qemu.weilnetz.de/w64/>) and add
   its folder to `PATH`.
2. Enable the **Windows Hypervisor Platform** feature
   (*Windows Features → "Windows Hypervisor Platform"*) for acceleration via WHPX.
3. Launch the disk:

```bat
qemu-system-x86_64.exe ^
  -accel whpx ^
  -m 4096 -smp 4 ^
  -drive file=C:\path\to\rosetta.qcow2,if=virtio,format=qcow2 ^
  -netdev user,id=n0 -device virtio-net,netdev=n0 ^
  -display sdl
```

If WHPX fails to initialize, drop `-accel whpx` to fall back to software emulation
(much slower but functional).

> **Firmware (all hosts):** QEMU's default firmware (SeaBIOS) is BIOS, so do **not** add
> OVMF/UEFI (`-bios OVMF.fd` / `pflash`). In Virt-manager, leave firmware as "BIOS." Disk
> bus should be **virtio** (AHCI/SATA also work, since those drivers are built into the
> kernel).

---

### Editing Rosetta from the Companion OS (Ubuntu) via chroot

In **side-by-side mode**, the way to inspect or modify Rosetta's filesystem offline is to
boot the companion OS (Ubuntu), mount Rosetta's partition, mount the virtual kernel
filesystems, and `chroot` in — the same mechanism used to build it.

> In **standalone mode** there is no companion OS to chroot from: boot Rosetta directly
> and edit live, or use a live USB. (You can also edit the disk image offline from the
> hypervisor host with `qemu-nbd` — see the note at the end of this section.)

**1. Boot the VM and select Ubuntu** at the GRUB menu, then become root:

```bash
sudo -i
```

**2. Identify Rosetta's root partition.** Do not assume a device name — verify:

```bash
lsblk -o NAME,SIZE,FSTYPE,UUID,MOUNTPOINT
```

Rosetta's root is the ext4 partition that is *not* the running `/`. During development
this is `/dev/vda3` (Ubuntu is `/dev/vda2`). Confirm after mounting (step 4) that
`/etc/hostname` reads `Rosetta`.

**3. Mount the Rosetta partition:**

```bash
export LFS=/mnt/lfs
mkdir -pv $LFS
mount -v -t ext4 /dev/vda3 $LFS          # use the device you identified above
```

**4. Verify it's Rosetta before going further:**

```bash
cat $LFS/etc/hostname                    # must read: Rosetta
```

**5. Mount the virtual kernel filesystems** (required for `chroot`; without these,
package tools and many builds fail):

```bash
mount -v --bind /dev    $LFS/dev
mount -v --bind /dev/pts $LFS/dev/pts
mount -vt proc  proc    $LFS/proc
mount -vt sysfs sysfs   $LFS/sys
mount -vt tmpfs tmpfs   $LFS/run
if [ -h $LFS/dev/shm ]; then
  install -v -d -m 1777 $LFS$(realpath /dev/shm)
else
  mount -vt tmpfs -o nosuid,nodev tmpfs $LFS/dev/shm
fi
```

**6. Enter the chroot:**

```bash
chroot "$LFS" /usr/bin/env -i \
    HOME=/root TERM="$TERM" \
    PS1='(rosetta chroot) \u:\w\$ ' \
    PATH=/usr/bin:/usr/sbin \
    /bin/bash --login
```

Confirm you're in the right place: `whoami` → `root`, the prompt shows
`(rosetta chroot)`, and `cat /etc/hostname` → `Rosetta`. You can now edit Rosetta's
files, run its package/orchestration scripts, etc., as root.

**7. Exit and unmount cleanly** when done (reverse order; always unmount before powering
off or re-running):

```bash
exit
umount $LFS/dev/shm 2>/dev/null
umount $LFS/dev/pts
umount $LFS/dev
umount $LFS/run
umount $LFS/proc
umount $LFS/sys
umount $LFS
```

> **Reminder:** these virtual filesystems do **not** survive a reboot or exiting the
> companion OS. Always re-run step 5 before re-entering the chroot. A quick `df -h`
> inside the chroot (no *"cannot read table of mounted file systems"* warning) confirms
> they are mounted.

> **Editing from the hypervisor host instead:** if Rosetta is its own VM and you want to
> edit its `.qcow2` without booting it, attach the disk image on the libvirt host with
> `qemu-nbd` (the VM must be powered off first):
>
> ```bash
> sudo modprobe nbd
> sudo qemu-nbd --connect=/dev/nbd0 /path/to/rosetta.qcow2
> lsblk -o NAME,SIZE,FSTYPE,UUID /dev/nbd0      # Rosetta root is the 30G ext4 (nbd0p3)
> sudo mount /dev/nbd0p3 /mnt/lfs
> # ... edit, or mount the virtual filesystems (step 5) and chroot ...
> sudo umount /mnt/lfs
> sudo qemu-nbd --disconnect /dev/nbd0
> ```

---

## Updating the GRUB Entries

How GRUB is updated depends on which boot mode Rosetta is in.

First, gather the values any entry needs (run on the host or companion OS, with
Rosetta's disk attached):

```bash
lsblk -o NAME,UUID,PARTUUID,MOUNTPOINT          # filesystem UUID + partition PARTUUID
ls /mnt/lfs/boot/vmlinuz*                        # exact kernel filename
```

For the current development image these are:

| Value | Current image |
|---|---|
| Filesystem UUID (for `search --fs-uuid`) | `f227344e-0e35-4511-a38f-2d3bc5cc2590` |
| Partition PARTUUID (for `root=PARTUUID=`) | `009d2a0a-22e4-4dcc-9a67-6450a6447410` |
| Kernel image | `vmlinuz-6.18.10-rosetta-lfs-13.0-systemd` |

> **Re-fetch these for any rebuilt or cloned disk** — they will differ, and copying a
> stale value produces an unbootable entry.

### Side-by-side mode (companion OS manages the menu)

The companion's GRUB owns the boot menu. Edit its custom entries file and regenerate —
**on the companion OS (e.g. Ubuntu), not inside Rosetta:**

```bash
sudo nano /etc/grub.d/40_custom
```

Add or edit the entry (this is also where you rename the menu title — the text in quotes
is cosmetic):

```bash
menuentry "Rosetta" {
        insmod part_gpt
        insmod ext2
        search --set=root --fs-uuid f227344e-0e35-4511-a38f-2d3bc5cc2590
        linux /boot/vmlinuz-6.18.10-rosetta-lfs-13.0-systemd root=PARTUUID=009d2a0a-22e4-4dcc-9a67-6450a6447410 ro
}
```

Then regenerate the menu:

```bash
sudo update-grub
```

> Do **not** hand-edit the generated `/boot/grub/grub.cfg` directly — the next
> `update-grub` (which runs automatically on companion-OS kernel updates) overwrites it.
> Always edit `40_custom`.
>
> If you prefer the entry to be auto-detected by `os-prober` (it appears as *"unknown
> Linux distribution"*), enable detection with `GRUB_DISABLE_OS_PROBER=false` in
> `/etc/default/grub` before `update-grub`. A hand-written `40_custom` entry is more
> reliable than the auto-detected one.

### Standalone mode (Rosetta owns the boot record)

Rosetta hand-writes its own config (LFS does not use `update-grub` / `grub-mkconfig`).
Edit it **inside Rosetta** (or its chroot):

```bash
nano /boot/grub/grub.cfg
```

```bash
# Begin /boot/grub/grub.cfg
set default=0
set timeout=5
insmod part_gpt
insmod ext2
set gfxpayload=1024x768x32

menuentry "Rosetta" {
        search --set=root --fs-uuid f227344e-0e35-4511-a38f-2d3bc5cc2590
        linux /boot/vmlinuz-6.18.10-rosetta-lfs-13.0-systemd root=PARTUUID=009d2a0a-22e4-4dcc-9a67-6450a6447410 ro
}
# End /boot/grub/grub.cfg
```

Changes take effect on next reboot — there is no regeneration step. If you reinstalled
GRUB to a new disk, also run `grub-install /dev/<disk>` once so the boot record points at
this config.

> **Why UUID / PARTUUID:** `search --fs-uuid` lets GRUB locate the kernel by filesystem
> identity, and `root=PARTUUID=` lets the kernel mount root **without an initramfs**.
> Both are independent of the device name, so entries keep working whether the disk
> enumerates as `vda`, `sda`, or `nvme`. Do **not** use `root=UUID=` (filesystem UUID on
> the kernel line) — that requires an initramfs, which Rosetta intentionally does not
> have.

---

## Converting from Side-by-Side to Standalone

To make Rosetta own the boot process instead of relying on the companion's GRUB:

1. From within Rosetta (or a chroot into it), install GRUB to the disk:
   ```bash
   grub-install /dev/<disk>     # e.g. /dev/vda — writes to the BIOS-boot partition
   ```
   > ⚠️ This **overwrites** any existing bootloader on that disk (including the companion
   > OS's GRUB). Do this only on a disk/VM where losing the companion's boot menu is
   > acceptable — a cloned VM is ideal.
2. Write Rosetta's own `/boot/grub/grub.cfg` using `search --set=root --fs-uuid <root-fs-UUID>`
   and `root=PARTUUID=<root-partition-UUID>` (see *Standalone mode* above).
3. Reboot. The disk now boots Rosetta independently of any companion OS.

---

## Deploying to a New VM

1. Copy/convert the disk image, or rebuild a clean disk from the `.tar.xz` archive:
   ```bash
   # compact copy of the full image (keeps both partitions):
   qemu-img convert -O qcow2 -c source.qcow2 rosetta-new.qcow2
   ```
2. Create a new VM via **Import existing disk image**; set firmware to **BIOS** and disk
   bus to **virtio**.
3. Boot.

> ⚠️ **Cloned images share filesystem UUID/PARTUUID with their source.** Do not attach a
> clone and its source to the **same** VM simultaneously, or boot-time UUID resolution
> becomes ambiguous. Running them as **separate** VMs is fine.

---

## Known Constraints & Roadmap

- **BIOS only** — no UEFI support yet (needs an EFI System Partition + UEFI GRUB).
- **QEMU/KVM only** — partitioning and device-naming assumptions are tied to the
  development hypervisor. Other hypervisors are untested (would require image-format
  conversion and re-validation).
- **Wi-Fi** requires a BLFS supplicant before it associates.
- **Cross-platform (Windows/Mac) compatibility** — the core research goal; not yet
  implemented.
- **Compatibility table** — to be added once data is available.

---

## Quick Reference

| Item | Value |
|---|---|
| Base | LFS 13.0 (systemd) |
| Kernel | 6.18.10 (drivers built-in, no initramfs) |
| Hostname | `Rosetta` |
| Locale | `C.UTF-8` |
| Timezone | `America/Chicago` (RTC in UTC) |
| Firmware | Legacy BIOS (GPT + BIOS-boot partition) |
| Root FS | ext4 |
| Networking | DHCP, type-matched interface names, `systemd-resolved` |
| Boot resolution | `search --fs-uuid` + `root=PARTUUID=` |
| Hypervisor | QEMU/KVM (Virt-manager) |
| Companion OS | Ubuntu (tested) |
