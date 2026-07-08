# Image Workflow for Mac Pro 6,1

## Overview

ISO build assets and integration notes for a Mac Pro 6,1-oriented image flow. The committed implementation prepares an AnduinOS-based build tree, overlays Mac Pro defaults, and integrates the macOS Tahoe KVM toolkit. A fully custom kernel and Mesa stack are optional inputs, not guaranteed by the script unless you provide them.

| ISO | Base | Desktop | Installer | Target |
|-----|------|---------|-----------|--------|
| **AnduinOS** | Ubuntu LTS | GNOME (Windows-like UX) | Calamares | Stability and familiarity |

## What's Included

A prepared image workflow with optional custom components.

What this tree can provide:

- **Desktop environment** — GNOME, fully configured
- **Custom kernel** (`linux-macpro61`) — optional, if you pass `--kernel-deb` to the build script
- **Custom Mesa `.deb` set** with RADV Vulkan — optional, if you pass `--mesa-debs` to the build script
- **macOS Tahoe KVM** — desktop icon and one-click setup scripts, with recovery downloaded directly from Apple
- **OpenCore.qcow2** — optional, included only if present under `macos-tahoe-kvm/opencore-efi/`
- **Fan/sysctl defaults** — this tree ships sysctl overlay; fan control docs/config live in `configs/MacPro6,1/fan/`
- **Sysctl tuning** — kvm.ignore_msrs, network, memory optimizations
- **First-boot Tahoe launcher setup** — systemd oneshot added to the image overlay

The macOS recovery image (~700MB) downloads directly from Apple's servers when you click the desktop icon. The one-click setup implements the current toolkit flow: dependency checks, local OVMF staging, recovery download, disk creation, and launch-script generation. The manual OSX-KVM path remains separately documented in [`../docs/kvm-macos.md`](../docs/kvm-macos.md).

## Build

### AnduinOS (Ubuntu-based)

```bash
# Requires: AMD64 Linux host, debootstrap, live-build, squashfs-tools
cd image/anduinos
sudo ./build.sh
```

The committed `build.sh` is a preparation/orchestration script. It clones the AnduinOS build system, prepares overlays, stages optional `.deb` inputs, and prints the remaining manual build steps that happen inside the upstream AnduinOS tree.

**How it works:**
1. Clones AnduinOS build system (Ubuntu LTS + GNOME + Calamares)
2. Optionally injects a custom kernel package (`--kernel-deb`)
3. Optionally stages Mesa packages (`--mesa-debs`)
4. Copies macOS Tahoe KVM toolkit to /opt/macos-tahoe-kvm/
5. Installs desktop launcher (first-boot systemd oneshot)
6. Adds sysctl tuning overlay
7. Hands off to the upstream AnduinOS build flow


## Directory Structure

```
image/
├── README.md              # This file
├── common/                # Shared overlay/package inputs
│   ├── overlay/           # Filesystem overlay (sysctl, scripts, etc.)
│   │   ├── etc/
│   │   │   └── sysctl.d/
│   │   │       └── 99-macpro.conf
│   └── packages-common.txt
├── anduinos/
│   ├── build.sh           # AnduinOS ISO build script
```

## Build Dependencies

### AnduinOS build host
```
debootstrap live-build squashfs-tools xorriso mtools grub-efi-amd64-bin
```

## Release Workflow

1. Build or obtain a custom kernel `.deb` if you want the non-stock kernel
2. Build or obtain Mesa `.deb` packages if you want a non-stock Mesa stack
3. Run `cd image/anduinos && sudo ./build.sh [--kernel-deb ...] [--mesa-debs ...]`
4. Follow the manual build/remaster steps printed by the script inside the AnduinOS build tree
5. Test the resulting ISO in QEMU
6. Write to USB and boot Mac Pro 6,1 hardware
7. Install via Calamares and use full poweroff, not warm reboot, when switching kernels
