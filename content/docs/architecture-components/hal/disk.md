---
title: Disk-Backed HAL
---

## Overview

Cocos uses a disk-backed HAL flow for CVMs. Instead of running
the full guest environment entirely from an in-memory initramfs, the CVM boots
from a disk image that contains:

- an EFI system partition with GRUB
- a kernel and initramfs
- a read-only root filesystem partition protected by dm-verity
- a separate dm-verity hash partition
- a separate writable `/cocos` partition

The early initramfs is still part of the boot chain, it mounts the real root
filesystem read-only, provisions an encrypted `/cocos` volume, prepares a few
runtime mounts, and then switches into the installed system.

This complements the RAM-only HAL described in [On-premises](./hal.md). The
RAM-only path keeps everything in memory, while the disk-backed path gives the
guest a richer OS image plus an encrypted writable data area.

## High-Level Architecture

The current disk-backed HAL implementation is composed of these pieces:

1. **Buildroot external tree**: The HAL lives under `cocos/hal/disk` and builds
   the bootable image layout.
2. **EFI boot image**: GRUB loads the kernel and initramfs from the EFI system
   partition.
3. **Early initramfs**: The initramfs opens the dm-verity root mapping,
   provisions the encrypted `/cocos` partition, rewrites `/etc/fstab`, and runs
   `switch_root`.
4. **Runtime system**: The guest OS continues booting from the read-only root
   partition while using `/cocos` for writable state.
5. **Manager/QEMU integration**: The [Manager](../manager.md) can either boot a
   kernel and initramfs directly or boot from a disk image when disk mode is
   enabled.

## Disk Layout

The current HAL image produces a GPT disk with four main partitions:

- **EFI partition**: FAT filesystem containing GRUB, the kernel, and the
  initramfs
- **Root partition**: ext4 root filesystem used as the dm-verity data device
- **Verity partition**: dm-verity hash tree for the root filesystem
- **Cocos partition**: writable partition provisioned at boot as LUKS2 and
  mounted at `/cocos`

At runtime, the initramfs expects the boot disk as `/dev/sda` and uses:

- `/dev/sda2` as the root filesystem partition
- `/dev/sda3` as the verity hash partition
- `/dev/sda4` as the writable `/cocos` partition

## Boot Flow

The disk-backed initramfs runs as PID 1 and performs the early boot sequence.

### 1. Early System Setup

The initramfs initializes:

- `/dev`, `/proc`, `/sys`, `/run`, and `/tmp`
- `devtmpfs` and `devpts`
- `configfs` when available

### 2. Root Filesystem Verification And Mount

The initramfs:

1. assumes the main disk is `/dev/sda`
2. resolves the root partition as `/dev/sda2`
3. resolves the verity hash partition as `/dev/sda3`
4. reads the `roothash=` value from the kernel command line
5. opens `/dev/mapper/root_verity` with `veritysetup`
6. mounts `/dev/mapper/root_verity` read-only at `/root`

The root filesystem is not copied from elsewhere during boot. The VM boots
directly from the image already attached to QEMU, and dm-verity protects the
read-only root mount. The GRUB command line also disables late systemd verity
auto-setup because the initramfs has already opened the root mapping.

### 3. `/cocos` Provisioning

The initramfs then provisions the writable data partition:

1. resolves the writable partition as `/dev/sda4`
2. generates a fresh ephemeral key
3. formats `/dev/sda4` as a LUKS2 container
4. opens it as `/dev/mapper/cocos_crypt`
5. formats that mapper as ext4
6. mounts it at `/root/cocos`

It also prepares the expected directory structure on `/cocos`, including:

- `.cache/oci`
- `datasets`
- `docker`
- `cocos_init`

### 4. Writable Runtime Mounts

Because the root filesystem is intentionally read-only, the initramfs mounts:

- `tmpfs` on `/tmp`
- `tmpfs` on `/var`

It then redirects selected writable paths:

- `/var/lib/docker` is bind-mounted from `/cocos/docker`
- `/cocos_init` is backed by `/cocos/cocos_init`

### 5. Configuration Handoff

Before switching root, the initramfs rewrites `/etc/fstab` in the mounted root
filesystem so the live runtime view is consistent.

It also preserves or adds 9P mounts for:

- `certs_share` -> `/etc/certs`
- `env_share` -> `/etc/cocos`

### 6. Switch Root

After provisioning is complete, the initramfs securely wipes the temporary key
file and executes:

```bash
switch_root /root /sbin/init
```

The installed system then continues normal boot from the already mounted
read-only root.

## Runtime Filesystem Model

The running system is split into:

- read-only root on `/`
- encrypted writable storage on `/cocos`
- `tmpfs` on `/tmp`
- `tmpfs` on `/var`

This means service state is intentionally redirected away from the root
filesystem. In particular:

- Docker data lives under `/cocos/docker`
- agent working data lives under `/cocos`
- runtime helper scripts use `/cocos_init`

From the guest’s point of view, `/cocos` behaves like a normal directory tree,
but it is backed by an encrypted mapper created during early boot.

## Security Model

The current design has a few important properties:

- the root filesystem is mounted read-only
- the root filesystem is protected by dm-verity for integrity
- `/cocos` is protected by LUKS2 using an ephemeral per-boot key
- that key is wiped before `switch_root`
- `/etc/certs` and `/etc/cocos` are supplied separately through 9P mounts

This means the protections are split by purpose:

- **Root filesystem**: integrity-protected and read-only
- **`/cocos`**: encrypted and writable
- **`/tmp` and `/var`**: volatile `tmpfs`

This is not a persistent disk unlock scheme. The writable `/cocos` partition is
provisioned fresh on each boot using a key that is not retained for later
reboots.

## Manager Integration

The Manager participates in this flow in two different ways:

- **Direct kernel/initramfs boot**: the Manager passes `-kernel` and `-initrd`
  to QEMU for RAM-only or direct-boot workflows
- **Disk boot**: when `MANAGER_QEMU_ENABLE_DISK=true`, the Manager boots from an
  attached disk image instead of passing `-kernel` and `-initrd`

In disk mode, the disk artifact must already contain the EFI boot chain, kernel,
initramfs, and root filesystem expected by this HAL flow.

See the [Manager](../manager.md) page for the Manager-side environment
variables, disk boot settings, and launch examples.

## Build Artifacts

The HAL build produces:

- `disk.img`: the bootable GPT disk image
- `rootfs.ext4`: the root filesystem image
- `rootfs.verity`: the dm-verity hash image for the root filesystem
- `rootfs.roothash`: the dm-verity root hash embedded into the GRUB command line
- `rootfs.cpio.gz`: the early initramfs used during boot

These artifacts come from the disk HAL Buildroot workflow under `cocos/hal/disk`
rather than the older `cloud-init`-based source-image copy flow.
