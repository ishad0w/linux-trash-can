# Repository Map

This file is the navigation index for `linux-mac`: what each area does, how the build paths differ, and where to start when changing a specific subsystem.

## Read This First

| File | Use it for |
|------|------|
| [`README.md`](README.md) | Project overview, hardware support matrix, and primary documentation index. |
| [`AGENTS.md`](AGENTS.md) | Working rules, source-of-truth notes, sync rules, and validation expectations. |
| [`configs/MacPro6,1/README.md`](configs/MacPro6,1/README.md) | The Mac Pro 6,1 hardware matrix and model-specific assumptions. |
| [`configs/MacPro6,1/TRIM-RULES.md`](configs/MacPro6,1/TRIM-RULES.md) | What kernel subsystems are protected vs safe to trim. |
| [`image/README.md`](image/README.md) | ISO-level architecture and the AnduinOS image workflow. |
| [`macos-tahoe-kvm/ISO-INTEGRATION.md`](macos-tahoe-kvm/ISO-INTEGRATION.md) | How the Tahoe KVM toolkit is supposed to be embedded into an ISO. |
| [`docs/kvm-macos.md`](docs/kvm-macos.md) | Manual macOS Tahoe on KVM walkthrough. |
| [`docs/gpu-acceleration.md`](docs/gpu-acceleration.md) | GPU stack, feature matrix, and tuning guidance. |
| [`docs/mesa.md`](docs/mesa.md) | Mesa userspace setup and troubleshooting. |
| [`docs/pvg-linux.md`](docs/pvg-linux.md) | Long-term GPU virtualization roadmap. |

## How The Repository Fits Together

```text
configs/MacPro6,1/config-stock-6.19
    -> configs/MacPro6,1/config
    -> scripts/build.sh

packaging/arch/config + packaging/arch/*.patch
    -> packaging/arch/PKGBUILD
    -> Arch package artifacts

image/common/* + macos-tahoe-kvm/*
    -> image/anduinos/build.sh
    -> preconfigured ISO workflow

docs/* + README.md
    -> explain the runtime model, hardware assumptions, and future work
```

## Current Change Surfaces

| Surface | Main files | What is really changing here |
|------|------|------|
| Stock-to-model config trim | [`configs/MacPro6,1/config-stock-6.19`](configs/MacPro6,1/config-stock-6.19), [`configs/MacPro6,1/config`](configs/MacPro6,1/config) | The only committed stock baseline is historical `6.19`. It has `3205` built-ins, `6227` modules, and `875` disabled options. The trimmed model config drops this to `2607` built-ins, `1` remaining module, and `3024` disabled options; the last module is `CONFIG_DRM_AMDGPU=m`. Use the delta as scale, not as a same-version proof. |
| Packaged kernel seed config | [`packaging/arch/config`](packaging/arch/config), [`packaging/arch/PKGBUILD`](packaging/arch/PKGBUILD) | The package config is more modular on disk (`2638` built-ins, `3608` modules, `4161` disabled), then `prepare()` embeds firmware, applies patches, enables BORE, and force-flips critical options to `=y`. This is also the only path that applies the active CachyOS patch stack and it targets `7.0rc1`, not the raw builder's default `6.19` path. |
| Active patch stack | [`packaging/arch/0001-bore.patch`](packaging/arch/0001-bore.patch) to [`packaging/arch/0005-hdmi.patch`](packaging/arch/0005-hdmi.patch) | BORE scheduler, BBR3 TCP, CachyOS performance features, follow-up fixes, and AMD display/HDMI updates. |
| Reference patch archive | [`patches/cachyos/`](patches/cachyos) | Disabled/reference copies of the CachyOS patch set. They are useful for comparison, but they are not the packaging source of truth and they are not all identical to the active stack. |
| Runtime tuning | [`configs/MacPro6,1/sysctl.d/99-macpro.conf`](configs/MacPro6,1/sysctl.d/99-macpro.conf), [`packaging/arch/99-macpro.conf`](packaging/arch/99-macpro.conf), [`image/common/overlay/etc/sysctl.d/99-macpro.conf`](image/common/overlay/etc/sysctl.d/99-macpro.conf), [`configs/MacPro6,1/fan/macfanctld.conf`](configs/MacPro6,1/fan/macfanctld.conf) | Sysctl defaults are duplicated for the config, package, and ISO paths. Fan behavior lives separately as an `applesmc`/`macfanctld` policy file. |
| macOS Tahoe workflows | [`scripts/launch-macos.sh`](scripts/launch-macos.sh), [`macos-tahoe-kvm/scripts/`](macos-tahoe-kvm/scripts), [`docs/kvm-macos.md`](docs/kvm-macos.md) | There is a documented manual KVM path and a generated one-click toolkit path. Both describe the same guest model with different entry points. |
| KVM artifact layouts | [`scripts/launch-macos.sh`](scripts/launch-macos.sh), [`macos-tahoe-kvm/scripts/boot-local.sh`](macos-tahoe-kvm/scripts/boot-local.sh), [`macos-tahoe-kvm/scripts/boot-vnc.sh`](macos-tahoe-kvm/scripts/boot-vnc.sh), [`macos-tahoe-kvm/scripts/macos-tahoe-setup.sh`](macos-tahoe-kvm/scripts/macos-tahoe-setup.sh) | The launchers do not use one shared file layout or one shared QEMU contract: top-level script expects `$HOME/kvm` and `BaseSystem.dmg`; `boot-local.sh` and `boot-vnc.sh` expect an OSX-KVM-style tree and `BaseSystem.img` with `-cpu host` plus QXL; the setup toolkit generates `vm/recovery/BaseSystem.dmg`, copies local OVMF, and emits a launcher using `Haswell-noTSX` plus `-vga std`. |

## File Map

### Root

| Path | Purpose | Notes |
|------|------|------|
| [`README.md`](README.md) | Top-level project description, support matrix, quick start, and docs hub. | Reads from the packaged-kernel point of view; use this map for raw-vs-packaged build divergences. Also records that the original GitHub upstreams were archived on March 10, 2026 and that this repo is now the live source of truth. |
| [`CONTRIBUTING.md`](CONTRIBUTING.md) | Contribution guide for adding new Mac models and improving existing configs. | Uses `configs/MacPro6,1` as the template. |
| [`AGENTS.md`](AGENTS.md) | Working rules for humans and automation. | Added to make future repo work less assumption-driven. |
| [`TECH-DEBT.md`](TECH-DEBT.md) | Current debt register for stale, partial, or structurally divergent repo paths. | Start here when deciding what to delete, unify, or deprecate next. |
| [`MAP.md`](MAP.md) | File-by-file repository map. | This file. |
| [`chatgpt-kernel-debug-prompt.md`](chatgpt-kernel-debug-prompt.md) | Reusable debug prompt for kernel troubleshooting on the Mac Pro 6,1. | Documents known constraints and a good issue-report template. |
| [`LICENSE`](LICENSE) | Licensing split. | Kernel configs/patches are GPL-2.0-only; scripts/docs are MIT. |

### `benchmarks/`

| Path | Purpose | Notes |
|------|------|------|
| [`benchmarks/baseline-6.19-stock.txt`](benchmarks/baseline-6.19-stock.txt) | Captured benchmark and system baseline for a stock 6.19 kernel before customization. | Useful as raw evidence, but the header and captured hardware details should be re-verified before quoting as canonical metadata. |

### `scripts/`

| Path | Purpose | Notes |
|------|------|------|
| [`scripts/build.sh`](scripts/build.sh) | Generic raw kernel builder for `configs/<model>/config`. | Defaults to `6.19`, downloads a vanilla kernel.org tarball, copies the config, applies only model-local patches, and compiles with the host's default `make` toolchain. It does not perform the PKGBUILD fixups or apply the active CachyOS patch stack. |
| [`scripts/audit-modules.py`](scripts/audit-modules.py) | Helper for converting `=m` options into either `=y` or disabled states. | Encodes the repo's "near-zero-module workstation kernel" intent as regex-based rules. |
| [`scripts/launch-macos.sh`](scripts/launch-macos.sh) | Lightweight manual QEMU launcher for Tahoe on this kernel. | Expects `$HOME/kvm/{OpenCore.qcow2,macos-tahoe.qcow2,BaseSystem.dmg}` plus system OVMF under `/usr/share/edk2/x64`. Separate from the richer `macos-tahoe-kvm` toolkit and not the same manual flow documented in [`docs/kvm-macos.md`](docs/kvm-macos.md). |

### `configs/MacPro6,1/`

| Path | Purpose | Notes |
|------|------|------|
| [`configs/MacPro6,1/README.md`](configs/MacPro6,1/README.md) | Hardware matrix, supported features, and known issues for the Mac Pro 6,1. | Main model-specific reference, now explicitly distinguishing the raw model config from the packaged Arch path. |
| [`configs/MacPro6,1/TRIM-RULES.md`](configs/MacPro6,1/TRIM-RULES.md) | Detailed keep/remove policy for trimming a generic kernel config to this machine. | Important before changing large config sections. |
| [`configs/MacPro6,1/config-stock-6.19`](configs/MacPro6,1/config-stock-6.19) | Stock Linux 6.19 baseline config captured before trimming. | Historical reference input for understanding scale and intent; not a same-version mirror of the current `7.0rc1` package path. |
| [`configs/MacPro6,1/config`](configs/MacPro6,1/config) | Trimmed Mac Pro 6,1 kernel config. | Hardware-focused config used by `scripts/build.sh`. It is not identical to the packaged build path and still leaves `CONFIG_DRM_AMDGPU=m`. |
| [`configs/MacPro6,1/patches/README.md`](configs/MacPro6,1/patches/README.md) | Placeholder documentation for future model-specific patches. | The directory is currently reserved, not populated. |
| [`configs/MacPro6,1/sysctl.d/99-macpro.conf`](configs/MacPro6,1/sysctl.d/99-macpro.conf) | Runtime sysctl defaults for memory, network, log noise, and KVM. | One of three copies of the same logical tuning policy. |
| [`configs/MacPro6,1/fan/macfanctld.conf`](configs/MacPro6,1/fan/macfanctld.conf) | Fan curve and thermal notes for the single centrifugal fan. | Documents the `applesmc` sensor path and a safe temperature ramp. |

### `docs/`

| Path | Purpose | Notes |
|------|------|------|
| [`docs/kvm-macos.md`](docs/kvm-macos.md) | Manual guide for running macOS Tahoe in QEMU/KVM. | Documents the manual OSX-KVM-style path, not the generated `macos-tahoe-kvm` toolkit layout. |
| [`docs/gpu-acceleration.md`](docs/gpu-acceleration.md) | Full GPU stack explanation for Linux on the FirePro D300/D500/D700. | Main feature matrix for OpenGL, Vulkan, VA-API, VCE, and rusticl. |
| [`docs/mesa.md`](docs/mesa.md) | Mesa userspace setup guide. | Focuses on driver packages, environment variables, and dual-GPU use. |
| [`docs/pvg-linux.md`](docs/pvg-linux.md) | PVG host roadmap for accelerating macOS guests on Linux. | Strategic design document, not an implemented subsystem. |

### `image/`

| Path | Purpose | Notes |
|------|------|------|
| [`image/README.md`](image/README.md) | Explains the preconfigured ISO strategy and directory layout. | The image-side hub document. |
| [`image/common/packages-common.txt`](image/common/packages-common.txt) | Common package list for any future Mac Pro ISO. | Captures the baseline KVM/QEMU, OVMF, and hardware-support package set without pinning one Mesa version. |
| [`image/common/overlay/etc/sysctl.d/99-macpro.conf`](image/common/overlay/etc/sysctl.d/99-macpro.conf) | ISO overlay copy of the runtime sysctl policy. | Includes `kvm.ignore_msrs = 1` for Tahoe guests. |
| [`image/anduinos/build.sh`](image/anduinos/build.sh) | Build-prep script for an AnduinOS-based installer ISO. | Clones upstream AnduinOS, injects overlays/tooling, and documents the final manual build step. |

### `macos-tahoe-kvm/`

| Path | Purpose | Notes |
|------|------|------|
| [`macos-tahoe-kvm/ISO-INTEGRATION.md`](macos-tahoe-kvm/ISO-INTEGRATION.md) | Contract for embedding the Tahoe toolkit into a Linux ISO. | Defines required files, first-boot integration, and optional GPU passthrough flow. Assumes the installed system provides an OVMF package. |
| [`macos-tahoe-kvm/scripts/macos-tahoe-setup.sh`](macos-tahoe-kvm/scripts/macos-tahoe-setup.sh) | Main end-user setup flow for Tahoe KVM. | Checks deps, detects hardware, copies OVMF from system packages, downloads recovery, creates disk, generates launch helpers, and optionally launches the VM. Uses `vm/recovery/BaseSystem.dmg` and expects `opencore-efi/OpenCore.qcow2`. |
| [`macos-tahoe-kvm/scripts/install-desktop-launcher.sh`](macos-tahoe-kvm/scripts/install-desktop-launcher.sh) | First-boot installer for the desktop icon and system launcher. | Also creates icon assets and optional prefetch behavior. |
| [`macos-tahoe-kvm/scripts/boot-local.sh`](macos-tahoe-kvm/scripts/boot-local.sh) | Simple GTK/local-display Tahoe boot helper. | Assumes a pre-existing local OSX-KVM-style directory layout with `OpenCore/OpenCore.qcow2`, `BaseSystem.img`, and local OVMF files. |
| [`macos-tahoe-kvm/scripts/boot-vnc.sh`](macos-tahoe-kvm/scripts/boot-vnc.sh) | Simple VNC-mode Tahoe boot helper. | Same artifact layout assumptions as `boot-local.sh`; daemonizes QEMU and exposes VNC on port `5901`. |

### `packaging/arch/`

| Path | Purpose | Notes |
|------|------|------|
| [`packaging/arch/PKGBUILD`](packaging/arch/PKGBUILD) | Active Arch packaging recipe for `linux-macpro61`. | Applies the full CachyOS-derived patch stack, stages firmware, enforces critical built-ins, uses `clang LLVM=1` plus Ivy Bridge `KCFLAGS`, and builds kernel plus headers packages. |
| [`packaging/arch/linux-macpro61.install`](packaging/arch/linux-macpro61.install) | Post-install hook. | Syncs the kernel to the ESP, masks reboot, adds a shell alias pushing users toward `poweroff`, and tolerates an initramfs if one exists even though the package path is designed to boot without one. |
| [`packaging/arch/config`](packaging/arch/config) | Seed kernel config for the Arch package path. | More general and more modular than the trimmed model config; final values are further changed in `prepare()`. |
| [`packaging/arch/99-macpro.conf`](packaging/arch/99-macpro.conf) | Package-installed sysctl defaults. | Package-path copy of the same tuning policy. |
| [`packaging/arch/0001-bore.patch`](packaging/arch/0001-bore.patch) | Adds the BORE scheduler. | Touches `kernel/sched/*`, Kconfig, and task structures. |
| [`packaging/arch/0002-bbr3.patch`](packaging/arch/0002-bbr3.patch) | Updates TCP congestion control to BBR3. | Mostly `net/ipv4/*` and TCP headers. |
| [`packaging/arch/0003-cachy.patch`](packaging/arch/0003-cachy.patch) | Main CachyOS tuning patch. | Adds ADIOS, `v4l2loopback`, `vhba`, `intel-nvme-remap`, and multiple scheduler/mm/AMD display tweaks. |
| [`packaging/arch/0004-fixes.patch`](packaging/arch/0004-fixes.patch) | Follow-up fixes patch. | Scheduler/mm/TLB cleanup plus smaller bluetooth, USB, i915, and HDA fixes. |
| [`packaging/arch/0005-hdmi.patch`](packaging/arch/0005-hdmi.patch) | AMD display/HDMI enhancement patch. | Adds HDMI VRR/ALLM-related display changes; there is no disabled mirror for this file. |

### `patches/cachyos/`

| Path | Purpose | Notes |
|------|------|------|
| [`patches/cachyos/0001-bore.patch.disabled`](patches/cachyos/0001-bore.patch.disabled) | Reference copy of the BORE patch. | File set matches the active BORE patch, but the blob is not stored as the active source of truth. |
| [`patches/cachyos/0002-bbr3.patch.disabled`](patches/cachyos/0002-bbr3.patch.disabled) | Reference copy of the BBR3 patch. | Diff scope is not identical to the active packaging patch and also touches `net/ipv4/tcp_rate.c`. |
| [`patches/cachyos/0003-cachy.patch.disabled`](patches/cachyos/0003-cachy.patch.disabled) | Reference copy of the main CachyOS tuning patch. | Same broad subsystem coverage as the active packaging patch. |
| [`patches/cachyos/0004-fixes.patch.disabled`](patches/cachyos/0004-fixes.patch.disabled) | Reference copy of the fixes patch. | Diff scope is not identical to the active packaging fixes patch. |

## Generated And Runtime-Only Files

These are mentioned in docs but are not committed:

- `build/` from [`scripts/build.sh`](scripts/build.sh)
- `macos-tahoe-kvm/vm/` created by [`macos-tahoe-kvm/scripts/macos-tahoe-setup.sh`](macos-tahoe-kvm/scripts/macos-tahoe-setup.sh)
- `macos-tahoe-kvm/launch-macos-tahoe.sh` generated by the Tahoe setup script
- `macos-tahoe-kvm/scripts/find-iommu-groups.sh` generated by the Tahoe setup script
- `macos-tahoe-kvm/scripts/bind-vfio.sh` generated by the Tahoe setup script
- `macos-tahoe-kvm/opencore-efi/OpenCore.qcow2` expected input, not stored in git

## Current Nuances To Remember

- The repository has two kernel build paths and they do not start from the same config file.
- The repository's two build paths also default to different kernel-source versions: raw builder `6.19`, package path `7.0rc1`.
- The original GitHub upstreams `wolffcatskyy/linux-mac` and `wolffcatskyy/cachyos-macpro-iso` were archived on March 10, 2026. This repository is the maintained source of truth now.
- The package build path mutates config values during `prepare()`, so the checked-in `packaging/arch/config` is only a seed.
- The raw builder uses vanilla kernel.org sources and default host toolchain choices; the package path uses the active patch stack plus `clang LLVM=1` and Ivy Bridge-specific `KCFLAGS`.
- Existing top-level docs mostly describe the packaged kernel path. The raw model config is stricter and still differs on `CONFIG_DRM_AMDGPU`.
- Display support should be read together with the runtime guidance in [`docs/gpu-acceleration.md`](docs/gpu-acceleration.md): the working SI path uses `amdgpu.dc=0`, not the newer DC path.
- The active packaging patches and the disabled reference patches are related, but not all of them are identical.
- The repo says "no initramfs required", but the install hook still mirrors an initramfs to the ESP if one exists.
- The Tahoe workflow is documented in both a manual guide and an ISO-integrated toolkit; they should evolve together.
- The Tahoe launch helpers currently assume three different recovery/open-core layouts and at least two different CPU/display launch models, so any cleanup in that area needs to update docs and scripts together.

Start with [`AGENTS.md`](AGENTS.md) when planning work. Start with this file when you need to know where that work actually lives.
