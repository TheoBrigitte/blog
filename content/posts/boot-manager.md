---
title: "Managing Boot Rescue Elements with boot-manager"
date: 2025-10-01T23:15:00+02:00
description: "A bash script for backing up and managing systemd-boot, kernel, and initramfs"
---

When managing Linux systems with systemd-boot, kernel updates can sometimes leave your system unbootable. The `boot-manager` script provides a safety net by backing up your working boot configuration and making it easy to verify, restore, and update boot elements.

## What is boot-manager?

`boot-manager` is a bash-based tool that manages three critical boot rescue elements:
- **Bootloader** (systemd-boot)
- **Kernel**
- **Initramfs**

The script helps you create backups, verify their integrity, and update boot components while maintaining a known-good fallback configuration.

## Installation

The script follows a simple structure:
- Main script: `~/.local/bin/boot-manager`
- Libraries: `~/.local/lib/boot-manager/*.sh`
- Configuration: `~/.config/boot-manager/boot-manager.conf`

## Core Commands

### Status Check

The `status` command verifies your current boot configuration:

```bash
boot-manager status
```

This checks:
1. **systemd-boot** - Verifies the EFI entry is active and reads the version from `/boot/EFI/systemd-old/systemd-bootx64.efi`
2. **Boot entry** - Confirms the loader entry exists in systemd-boot
3. **Kernel** - Validates kernel and initramfs files match expected versions

Example output:
```
systemd-boot .......... 256.7
systemd-boot entry .... Linux Rescue
kernel ................ 6.6.103-2-MANJARO
```

### Backup

The `backup` command creates a complete backup of your boot configuration:

```bash
boot-manager backup
```

The backup process:
1. Exports EFI boot variables using `efibootmgr` and `efivar`
2. Backs up systemd-boot files from `/boot/EFI/systemd-old/`
3. Copies the loader entry, kernel, and initramfs
4. Generates SHA256 checksums for integrity verification

Key safety feature: The script refuses to run as root to prevent accidental system-wide changes.

### Verify Backup

Ensure your backup hasn't been corrupted:

```bash
boot-manager verify
```

This validates all checksums against the stored `sha256sums.txt` file.

### Configuration Management

Edit and validate your configuration:

```bash
# Edit configuration with validation
boot-manager config edit

# Validate current configuration
boot-manager config validate
```

Required configuration variables (`boot-manager.conf`):
- `SYSTEMD_DIR` - systemd-boot installation directory
- `SYSTEMD_OLD_DIR` - systemd-boot backup directory
- `KERNEL_VERSION` - Expected kernel version
- `LOADER_ENTRY` - Boot loader entry filename
- `BACKUP_DIR` - Backup storage location

## Script Architecture

The tool uses a modular design with separate library files:

```
~/.local/lib/boot-manager/
├── backup-boot.sh          # Backup operations
├── config.sh               # Configuration management
├── install-kernel.sh       # Kernel installation
├── status.sh               # Status checks
├── update-kernel.sh        # Kernel updates
├── update-systemd-boot.sh  # Bootloader updates
└── verify-backup.sh        # Backup verification
```

The main script (`boot-manager:24`) loads all libraries:
```bash
for f in "$LIB_DIR/"*.sh; do source $f; done
```

## Command Dispatch Pattern

The script uses a clever command dispatch pattern (`boot-manager:56-64`):

```bash
run_cmd() {
  local cmd="${1}_command"

  # Check if the command exists
  declare -F "$cmd" &> /dev/null || exit_error "unknown command"
  shift

  # Run the command
  "$cmd" "$@"
}
```

Each command is implemented as a `*_command` function (e.g., `status_command`, `backup_command`). This makes adding new commands straightforward - just create a new function following the naming convention.

## Use Cases

**Before a risky update:**
```bash
boot-manager backup
boot-manager verify
# Perform system update
boot-manager status
```

**Regular maintenance:**
```bash
# Check current boot status
boot-manager status

# Update bootloader after systemd upgrade
boot-manager update-bootloader
```

**Kernel updates:**
```bash
boot-manager install-kernel
```

## Why systemd-old?

The script specifically manages a "systemd-old" directory alongside the current systemd-boot installation. This provides a verified fallback bootloader version that can boot your system even if a systemd package update breaks the primary bootloader.

The status check (`status.sh:16`) verifies the old bootloader by reading its version string:
```bash
systemd_boot_version="$(strings /boot/EFI/systemd-old/systemd-bootx64.efi | \
  grep -m1 '#### LoaderInfo: systemd-boot' | awk '{print $4}')"
```

## Conclusion

`boot-manager` provides a structured approach to managing boot configurations on systemd-boot systems. By automating backups, verification, and status checks, it reduces the risk of unbootable systems after updates.

The modular design makes it easy to extend with additional commands, and the configuration management ensures critical variables are always set before operations run.

For systems where boot reliability is critical, having a scripted backup and verification process is essential - and `boot-manager` delivers exactly that.
