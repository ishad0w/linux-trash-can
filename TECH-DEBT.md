# Technical Debt

Current repo-level debt inventory for `linux-mac`.

Scope of this file:

- Tracks debt that is visible from the committed tree, current docs, and executable paths
- Focuses on things that are stale, partial, misleading, duplicated, or likely non-working without extra manual context
- Uses the current repository state as of 2026-04-22

This is not a full hardware validation report. Some items are confirmed by direct tree inspection and path mismatch, not by live boot/build testing.

## Priority Legend

- `P1` — structural debt or likely breakage that makes the repo hard to reason about
- `P2` — partial or stale paths that waste contributor time
- `P3` — lower-risk cleanup and alignment work

## P1

### TD-001 — Two kernel build systems produce materially different kernels

**State:** open  
**Area:** kernel build / packaging

Evidence:

- [`scripts/build.sh`](scripts/build.sh) defaults to `6.19`, downloads vanilla kernel.org tarballs, applies only `configs/<model>/patches/*.patch`, and builds with the host's plain `make`
- [`configs/MacPro6,1/patches/`](configs/MacPro6,1/patches/) currently contains only [`README.md`](configs/MacPro6,1/patches/README.md), so the raw builder applies no active feature patch stack for this model
- [`packaging/arch/PKGBUILD`](packaging/arch/PKGBUILD) targets `7.0rc1`, applies the active CachyOS-derived patch stack, uses `clang LLVM=1`, stages embedded firmware, and force-enables critical options
- [`configs/MacPro6,1/config`](configs/MacPro6,1/config) still keeps `CONFIG_DRM_AMDGPU=m`, while the package path forces `CONFIG_DRM_AMDGPU=y`

Why this matters:

- The repo describes one project, but today it has two incompatible kernel stories
- Support claims, performance claims, and bug reports can silently refer to different kernels
- New contributors can modify the wrong path and think they changed the shipped artifact

Exit criteria:

- Choose one primary build path and mark the other as legacy, or
- Make the raw builder consume the same version, patch stack, toolchain assumptions, and critical config fixups as the package path

### TD-002 — macOS Tahoe KVM tooling has three incompatible artifact layouts

**State:** open  
**Area:** KVM / VM launch scripts

Evidence:

- [`scripts/launch-macos.sh`](scripts/launch-macos.sh) expects `$HOME/kvm`, `BaseSystem.dmg`, `OpenCore.qcow2`, and OVMF under `/usr/share/edk2/x64`
- [`macos-tahoe-kvm/scripts/boot-local.sh`](macos-tahoe-kvm/scripts/boot-local.sh) and [`boot-vnc.sh`](macos-tahoe-kvm/scripts/boot-vnc.sh) expect an OSX-KVM-style local tree with `OpenCore/OpenCore.qcow2`, `BaseSystem.img`, and local OVMF files
- [`macos-tahoe-kvm/scripts/macos-tahoe-setup.sh`](macos-tahoe-kvm/scripts/macos-tahoe-setup.sh) generates `vm/recovery/BaseSystem.dmg`, `vm/ovmf/*`, and expects `opencore-efi/OpenCore.qcow2`
- [`docs/kvm-macos.md`](docs/kvm-macos.md) documents a `BaseSystem.dmg -> BaseSystem.img` conversion flow, while the generated toolkit keeps using `.dmg`
- [`docs/kvm-macos.md`](docs/kvm-macos.md) and the simple OSX-KVM-style helpers use `-cpu host` plus QXL, while the generated toolkit currently emits `Haswell-noTSX` plus `-vga std`
- [`scripts/launch-macos.sh`](scripts/launch-macos.sh) and the generated Tahoe toolkit both mount `BaseSystem.dmg` with `format=raw`, while the manual reference flow explicitly converts Apple recovery media to `BaseSystem.img` first

Why this matters:

- The helper scripts are not drop-in replacements for each other
- Some scripts are likely stale or non-working unless the user manually reshapes files into the expected layout
- The repo currently documents several mutually incompatible "correct" ways to boot Tahoe

Exit criteria:

- Standardize on one artifact layout and one recovery-image format
- Deprecate or delete stale launch helpers
- Make docs and scripts refer to the same directory contract

## P2

### TD-003 — The AnduinOS ISO builder is only a preparation script

**State:** open  
**Area:** image build / release flow

Evidence:

- [`image/anduinos/build.sh`](image/anduinos/build.sh) clones upstream, prepares overlays, stages optional packages, then prints manual build steps instead of producing the final ISO itself
- [`image/README.md`](image/README.md) now documents this correctly, but the implementation is still a prep/handoff workflow rather than a complete builder

Why this matters:

- Users expect a `build.sh` to produce the actual image, but here it stops at workspace preparation
- Release automation remains manual and hard to reproduce

Exit criteria:

- Either complete the end-to-end ISO build inside the repo script, or
- Rename/re-scope the script and documentation so it is clearly an image-prep step rather than a full builder

### TD-004 — The reference patch archive has drifted from the active patch stack

**State:** open  
**Area:** patch maintenance

Evidence:

- Active patches live in [`packaging/arch/`](packaging/arch/)
- Reference copies live in [`patches/cachyos/`](patches/cachyos/)
- At minimum, the BBR3 and fixes snapshots are no longer identical to the active packaging patches
- [`packaging/arch/0005-hdmi.patch`](packaging/arch/0005-hdmi.patch) has no mirror under `patches/cachyos/`

Why this matters:

- The repo exposes two patch sets that look related but are no longer trustworthy mirrors
- Rebases and patch reviews become ambiguous because there is no single visible source of truth

Exit criteria:

- Keep `patches/cachyos/` in sync automatically, or
- Move archival copies out of the active tree, or
- Delete the archive and treat `packaging/arch/*.patch` as the only source of truth

### TD-005 — `configs/MacPro6,1/patches/` is a placeholder branch with no implementation

**State:** open  
**Area:** model-specific patches

Evidence:

- [`configs/MacPro6,1/patches/README.md`](configs/MacPro6,1/patches/README.md) describes planned patches
- Both documented examples are explicitly "not yet written" / "maybe"
- The raw builder checks this directory, but for `MacPro6,1` it contributes no real patches

Why this matters:

- The directory looks active but is currently a dead end
- It suggests model-local patching exists when it does not

Exit criteria:

- Implement real model-local patches, or
- Remove the placeholder directory from the active raw-build path, or
- Move the ideas to issues/roadmap docs instead of leaving them in a live patch directory

### TD-006 — The no-initramfs design contract is still fuzzy

**State:** open  
**Area:** packaging / boot behavior

Evidence:

- Top-level docs describe the package path as booting without initramfs
- [`packaging/arch/linux-macpro61.install`](packaging/arch/linux-macpro61.install) still copies `initramfs-linux-macpro61.img` to the ESP if it exists

Why this matters:

- The repo simultaneously claims initramfs-free boot and keeps a fallback path alive
- Users and maintainers do not have a crisp answer to whether initramfs is supported, tolerated, or legacy baggage

Exit criteria:

- Decide whether initramfs is supported fallback or dead path
- Remove the contradictory branch or document it as intentional compatibility behavior

## P3

### TD-007 — Historical baselines are still doing work as live reference points

**State:** open  
**Area:** configs / benchmarks / expectations

Evidence:

- [`configs/MacPro6,1/config-stock-6.19`](configs/MacPro6,1/config-stock-6.19) is the only stock baseline in-tree
- [`benchmarks/baseline-6.19-stock.txt`](benchmarks/baseline-6.19-stock.txt) is a historical captured benchmark file
- The active package path targets `7.0rc1`

Why this matters:

- Historical baselines are still useful, but they are easy to overread as current evidence
- Version drift weakens config comparisons and performance claims

Exit criteria:

- Capture a same-generation baseline for the current package target, or
- Keep the historical files but label them as archival snapshots everywhere they are referenced

## Suggested Order

1. Resolve `TD-001` and `TD-002` first. They are the main reasons the repo is hard to reason about.
2. Then decide whether `TD-003`, `TD-004`, and `TD-005` stay as active surfaces or get archived/removed.
3. Finish with `TD-006` and `TD-007` to align contracts and reduce documentation drift.

## Explicitly Not Listed Here

These are known constraints, but they are not repo debt by themselves:

- Cold-boot requirement on Apple EFI
- Disabled sleep/hibernation on this hardware
- Need for external OpenCore artifacts that cannot be shipped in git
- KVM `ignore_msrs` requirement for macOS guests
