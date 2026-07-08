# Technical Debt

Current repo-level debt inventory for `linux-mac`.

Scope of this file:

- Tracks debt that is visible from the committed tree, current docs, and executable paths
- Focuses on things that are stale, partial, misleading, duplicated, or likely non-working without extra manual context
- Uses the current repository state as of 2026-05-04

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
- [`packaging/arch/PKGBUILD`](packaging/arch/PKGBUILD) targets CachyOS `7.1.3`, uses the CachyOS release tarball plus upstream 7.1 patchsource, stages embedded firmware, and force-enables critical options
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
- [`scripts/launch-macos.sh`](scripts/launch-macos.sh) also hardcodes the system OVMF location to `/usr/share/edk2/x64`, fixed SPICE port `5930`, and the default VM state directory to `$HOME/kvm`
- [`macos-tahoe-kvm/scripts/boot-vnc.sh`](macos-tahoe-kvm/scripts/boot-vnc.sh) uses a fixed monitor socket at `/tmp/qemu-monitor.sock`
- [`macos-tahoe-kvm/scripts/macos-tahoe-setup.sh`](macos-tahoe-kvm/scripts/macos-tahoe-setup.sh) can generate an `OPENCORE_IMAGE_PATH_HERE` placeholder into the launcher when OpenCore is absent

Why this matters:

- The helper scripts are not drop-in replacements for each other
- Some scripts are likely stale or non-working unless the user manually reshapes files into the expected layout
- The repo currently documents several mutually incompatible "correct" ways to boot Tahoe

Exit criteria:

- Standardize on one artifact layout and one recovery-image format
- Deprecate or delete stale launch helpers
- Make docs and scripts refer to the same directory contract
- Move VM paths, OVMF discovery, SPICE/VNC ports, and monitor socket paths behind one documented config surface

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

### TD-004 — Local carry patches need a 7.1.3 port audit

**State:** open
**Area:** patch maintenance

Evidence:

- Active patch sources are now the CachyOS `7.1.3` release tarball plus upstream `kernel-patches/master/7.1`
- Local snapshots still live in [`packaging/arch/`](packaging/arch/) and are documented in [`packaging/arch/PATCHES.md`](packaging/arch/PATCHES.md)
- Dry-run testing shows `0001-bore.patch`, much of `0002-bbr3.patch`, much of `0003-cachy.patch`, and many `0005-hdmi.patch` hunks are already present or superseded in the 7.1.3 source stack
- `0004-fixes.patch` and the failed portions of `0005-hdmi.patch` still need an explicit port audit before they can be re-enabled

Why this matters:

- The repo must not silently lose Mac Pro-relevant behavior just because the upstream base moved
- Rebases and patch reviews need a visible answer for each retained carry patch: active, superseded, or needs port

Exit criteria:

- Finish a hunk-by-hunk 7.1.3 audit for `0004-fixes.patch` and `0005-hdmi.patch`
- Either port the still-needed hunks into clean 7.1.3 carry patches or mark them superseded with source evidence
- Keep [`packaging/arch/PATCHES.md`](packaging/arch/PATCHES.md) current whenever the patch stack changes

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

### TD-008 — Runtime sysctl policy copies must stay aligned

**State:** open
**Area:** runtime tuning / KVM defaults

Evidence:

- [`configs/MacPro6,1/sysctl.d/99-macpro.conf`](configs/MacPro6,1/sysctl.d/99-macpro.conf) includes `kvm.ignore_msrs = 1`
- [`image/common/overlay/etc/sysctl.d/99-macpro.conf`](image/common/overlay/etc/sysctl.d/99-macpro.conf) includes the same KVM line
- [`packaging/arch/99-macpro.conf`](packaging/arch/99-macpro.conf), the copy installed by the Arch package path, now includes the KVM line
- Top-level KVM documentation tells users that `ignore_msrs=1` is required for macOS guests

Why this matters:

- Future edits can drift again because the same logical policy is copied in three places

Exit criteria:

- Keep `configs/MacPro6,1/sysctl.d/99-macpro.conf`, `packaging/arch/99-macpro.conf`, and `image/common/overlay/etc/sysctl.d/99-macpro.conf` synchronized
- Add a lightweight comparison check if this repo gains CI

### TD-009 — Install and desktop helpers mutate host-global paths without one policy

**State:** open
**Area:** packaging / host state / launcher install

Evidence:

- [`packaging/arch/linux-macpro61.install`](packaging/arch/linux-macpro61.install) syncs only to `/boot/efi`, masks `reboot.target`, and writes `/etc/profile.d/no-reboot.sh`
- [`macos-tahoe-kvm/scripts/install-desktop-launcher.sh`](macos-tahoe-kvm/scripts/install-desktop-launcher.sh) installs into `/opt/macos-tahoe-kvm`, `/usr/share/applications`, `/usr/share/icons`, and `/etc/systemd/system`, then scans `/etc/passwd` to copy desktop shortcuts
- The cold-boot policy is valid for this hardware, but today the package hook and desktop launcher make global changes without a shared config, dry-run mode, or explicit opt-out contract

Why this matters:

- Non-standard ESP layouts, non-desktop systems, or users who want the kernel without desktop/KVM helpers can get surprising global state changes
- Package install behavior and ISO integration behavior can drift because they encode similar policy in unrelated scripts

Exit criteria:

- Document supported ESP mount points and make the package hook configurable, or provide a clear failure message when `/boot/efi` is not the intended target
- Add an explicit opt-out or mode flag for global reboot masking and desktop launcher installation
- Keep package hook behavior, ISO overlay behavior, and desktop launcher behavior in one documented runtime policy

### TD-010 — External moving inputs are not pinned

**State:** open
**Area:** reproducibility / external downloads

Evidence:

- [`macos-tahoe-kvm/scripts/macos-tahoe-setup.sh`](macos-tahoe-kvm/scripts/macos-tahoe-setup.sh) downloads `macrecovery.py` from OpenCorePkg `master`
- The same setup path asks Apple recovery for `-os latest`
- [`image/anduinos/build.sh`](image/anduinos/build.sh) clones `AnduinOS.git` branch `main` and pulls the latest upstream state on repeated runs

Why this matters:

- A working build or KVM setup can change without any commit in this repository
- Debugging regressions becomes harder because the exact external input set is not recorded

Exit criteria:

- Pin external scripts/repos to reviewed commit hashes or release tags
- Record the selected macOS recovery catalog/version in generated state
- Add an update procedure for intentionally bumping external inputs

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

### TD-011 — Historical benchmark snapshot contains user-local paths and process noise

**State:** open
**Area:** evidence hygiene / privacy / reproducibility

Evidence:

- [`benchmarks/baseline-6.19-stock.txt`](benchmarks/baseline-6.19-stock.txt) includes captured process output with user-local paths such as `/home/michael/...`
- The same file records transient running processes and model paths that are not part of the kernel project contract
- The file is useful evidence, but it is not normalized or sanitized benchmark metadata

Why this matters:

- Readers can mistake incidental local workload state for project requirements
- User-specific paths make the artifact harder to reuse as a clean public reference

Exit criteria:

- Replace the raw capture with normalized benchmark metadata, or
- Keep the raw file but add a sanitized summary that is the only document used for support/performance claims

## Suggested Order

1. Resolve `TD-001`, `TD-002`, and `TD-009` first. They define which artifacts are built, which VM path is supported, and what the package is allowed to change on a host.
2. Then resolve `TD-003`, `TD-004`, `TD-005`, and `TD-010` so image prep, patch maintenance, placeholder paths, and external inputs become reproducible.
3. Finish with `TD-006`, `TD-007`, `TD-008`, and `TD-011` to align boot/sysctl contracts and clean historical evidence.

## Explicitly Not Listed Here

These are known constraints, but they are not repo debt by themselves:

- Cold-boot requirement on Apple EFI
- Disabled sleep/hibernation on this hardware
- Need for external OpenCore artifacts that cannot be shipped in git
- KVM `ignore_msrs` requirement for macOS guests
