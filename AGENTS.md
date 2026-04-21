# Working on linux-mac

`linux-mac` is a hardware-targeted kernel repository for the Mac Pro 6,1. It is not just a kernel config dump: it also carries packaging, runtime tuning, ISO integration, and a macOS Tahoe KVM workflow.

Use this file as the operating guide. Use [`MAP.md`](MAP.md) as the file-by-file index.

## Read Order

1. [`README.md`](README.md)
2. [`MAP.md`](MAP.md)
3. [`configs/MacPro6,1/README.md`](configs/MacPro6,1/README.md)
4. [`packaging/arch/PKGBUILD`](packaging/arch/PKGBUILD)
5. [`macos-tahoe-kvm/ISO-INTEGRATION.md`](macos-tahoe-kvm/ISO-INTEGRATION.md)

## Source Of Truth

| Topic | Primary files | Notes |
|------|------|------|
| Project scope and support matrix | [`README.md`](README.md) | Top-level story, hardware support, and main docs index. |
| Mac Pro 6,1 hardware profile | [`configs/MacPro6,1/README.md`](configs/MacPro6,1/README.md), [`configs/MacPro6,1/TRIM-RULES.md`](configs/MacPro6,1/TRIM-RULES.md) | Hardware matrix and trimming rules for this machine. |
| Raw kernel build path | [`scripts/build.sh`](scripts/build.sh), [`configs/MacPro6,1/config`](configs/MacPro6,1/config) | Copies the model config verbatim and runs `make olddefconfig`. |
| Arch package build path | [`packaging/arch/PKGBUILD`](packaging/arch/PKGBUILD), [`packaging/arch/config`](packaging/arch/config) | `prepare()` mutates the seed config, stages firmware, applies patches, and force-enables critical options. |
| Active patch stack | [`packaging/arch/0001-bore.patch`](packaging/arch/0001-bore.patch) to [`packaging/arch/0005-hdmi.patch`](packaging/arch/0005-hdmi.patch) | These are the patches the package build actually applies. |
| Reference CachyOS patch snapshots | [`patches/cachyos/`](patches/cachyos) | Reference copies only. They are not auto-applied and are not byte-identical to every active packaging patch. |
| Runtime tuning defaults | [`configs/MacPro6,1/sysctl.d/99-macpro.conf`](configs/MacPro6,1/sysctl.d/99-macpro.conf), [`packaging/arch/99-macpro.conf`](packaging/arch/99-macpro.conf), [`image/common/overlay/etc/sysctl.d/99-macpro.conf`](image/common/overlay/etc/sysctl.d/99-macpro.conf) | Same logical knobs live in three places. Keep them aligned. |
| KVM user flow | [`docs/kvm-macos.md`](docs/kvm-macos.md), [`scripts/launch-macos.sh`](scripts/launch-macos.sh), [`macos-tahoe-kvm/`](macos-tahoe-kvm) | There is both a top-level launcher path and a generated toolkit path. |
| ISO integration | [`image/README.md`](image/README.md), [`image/anduinos/build.sh`](image/anduinos/build.sh), [`image/common/`](image/common) | Overlay-based image workflow. |

## Hard Rules

- Keep new documentation in English. The existing repository docs are English-first and practical.
- Treat this repository as the maintained source of truth. The original upstream repos `wolffcatskyy/linux-mac` and `wolffcatskyy/cachyos-macpro-iso` were archived on March 10, 2026.
- Keep shell changes Bash-friendly and syntax-checkable with `bash -n`.
- Preserve the Mac Pro 6,1 invariants unless the change explicitly revisits them: cold boot requirement, Southern Islands GPU support, Apple SMC thermal path, `kvm.ignore_msrs`, and disabled sleep/hibernation.
- If you change a user-facing workflow, update both the docs and the executable script that implements it.
- If you change packaging inputs, update the active patch/config path and any reference copy that is meant to stay comparable.
- Do not assume a checked-in config file is the final shipped kernel. The packaging path rewrites part of the config during `prepare()`.
- When documenting GPU/display behavior, distinguish the raw model config from the packaged Arch path. They currently diverge on whether `amdgpu` is built-in, and the runtime guidance uses the SI legacy display path with `amdgpu.dc=0`.

## Important Nuances

- [`scripts/build.sh`](scripts/build.sh) and [`packaging/arch/PKGBUILD`](packaging/arch/PKGBUILD) do not build the same artifact. The raw builder uses [`configs/MacPro6,1/config`](configs/MacPro6,1/config) as-is. The PKGBUILD starts from [`packaging/arch/config`](packaging/arch/config), then mutates it before compile.
- [`scripts/build.sh`](scripts/build.sh) builds from vanilla kernel.org tarballs and applies only `configs/<model>/patches/*.patch`. For `MacPro6,1` that directory is currently empty, so the raw builder does not apply the active CachyOS/BORE/BBR3/HDMI patch stack at all.
- The two build paths are also on different kernel-source tracks by default: [`scripts/build.sh`](scripts/build.sh) defaults to `6.19` release tarballs from `cdn.kernel.org`, while [`packaging/arch/PKGBUILD`](packaging/arch/PKGBUILD) targets `7.0rc1` from Torvalds' git snapshot.
- [`configs/MacPro6,1/config`](configs/MacPro6,1/config) is the trimmed hardware profile. [`packaging/arch/config`](packaging/arch/config) is the package seed config. Neither file alone fully describes the final packaged kernel.
- The two build paths also use different toolchains and flags: the raw builder just runs `make`, while the package path uses `clang LLVM=1`, `ccache`, and `KCFLAGS="-march=ivybridge -O3 -pipe"`.
- The raw model config still has one module left: `CONFIG_DRM_AMDGPU=m`. Existing top-level docs mostly describe the packaged path, where `PKGBUILD` forces `CONFIG_DRM_AMDGPU=y`.
- Existing display support text should be interpreted through the boot/runtime guidance in [`docs/gpu-acceleration.md`](docs/gpu-acceleration.md): Southern Islands runs with `amdgpu.dc=0`, so this repo's working path is the legacy display path rather than DC.
- The active patch stack lives under [`packaging/arch/`](packaging/arch). The disabled snapshots under [`patches/cachyos/`](patches/cachyos) are archival/reference material and currently differ from the active stack for at least the BBR3 and fixes patch sets.
- Runtime tuning is duplicated on purpose so the same defaults can ship through the local config path, the package path, and the ISO overlay path. Change all three copies together.
- KVM guidance is split between the generic top-level launcher in [`scripts/launch-macos.sh`](scripts/launch-macos.sh) and the end-user toolkit in [`macos-tahoe-kvm/scripts/`](macos-tahoe-kvm/scripts). Keep them logically aligned.
- The KVM launchers do not share one artifact layout. [`scripts/launch-macos.sh`](scripts/launch-macos.sh) expects `$HOME/kvm` plus `BaseSystem.dmg`; [`macos-tahoe-kvm/scripts/boot-local.sh`](macos-tahoe-kvm/scripts/boot-local.sh) and [`macos-tahoe-kvm/scripts/boot-vnc.sh`](macos-tahoe-kvm/scripts/boot-vnc.sh) expect an OSX-KVM-style tree plus `BaseSystem.img`; [`macos-tahoe-kvm/scripts/macos-tahoe-setup.sh`](macos-tahoe-kvm/scripts/macos-tahoe-setup.sh) generates `vm/recovery/BaseSystem.dmg`.
- The Tahoe launch flows also diverge in launch defaults. [`docs/kvm-macos.md`](docs/kvm-macos.md) and the simple OSX-KVM-style helpers use `-cpu host` plus QXL and `BaseSystem.img`, while [`macos-tahoe-kvm/scripts/macos-tahoe-setup.sh`](macos-tahoe-kvm/scripts/macos-tahoe-setup.sh) currently generates a local `vm/` layout with `Haswell-noTSX`, `-vga std`, and `BaseSystem.dmg`.

## Change Matrix

| Change type | Minimum files to inspect | Usually also update |
|------|------|------|
| Kernel option changes | [`configs/MacPro6,1/config`](configs/MacPro6,1/config), [`packaging/arch/config`](packaging/arch/config), [`configs/MacPro6,1/TRIM-RULES.md`](configs/MacPro6,1/TRIM-RULES.md) | [`README.md`](README.md), [`configs/MacPro6,1/README.md`](configs/MacPro6,1/README.md), [`MAP.md`](MAP.md) |
| Patch stack changes | [`packaging/arch/*.patch`](packaging/arch), [`packaging/arch/PKGBUILD`](packaging/arch/PKGBUILD) | [`patches/cachyos/`](patches/cachyos), [`README.md`](README.md), [`MAP.md`](MAP.md) |
| Packaging and install-hook changes | [`packaging/arch/PKGBUILD`](packaging/arch/PKGBUILD), [`packaging/arch/linux-macpro61.install`](packaging/arch/linux-macpro61.install), [`packaging/arch/99-macpro.conf`](packaging/arch/99-macpro.conf) | [`README.md`](README.md), [`configs/MacPro6,1/sysctl.d/99-macpro.conf`](configs/MacPro6,1/sysctl.d/99-macpro.conf), [`image/common/overlay/etc/sysctl.d/99-macpro.conf`](image/common/overlay/etc/sysctl.d/99-macpro.conf) |
| macOS KVM flow changes | [`docs/kvm-macos.md`](docs/kvm-macos.md), [`scripts/launch-macos.sh`](scripts/launch-macos.sh), [`macos-tahoe-kvm/ISO-INTEGRATION.md`](macos-tahoe-kvm/ISO-INTEGRATION.md), [`macos-tahoe-kvm/scripts/`](macos-tahoe-kvm/scripts) | [`image/README.md`](image/README.md), [`image/anduinos/build.sh`](image/anduinos/build.sh) |
| ISO image changes | [`image/README.md`](image/README.md), [`image/anduinos/build.sh`](image/anduinos/build.sh), [`image/common/`](image/common) | [`macos-tahoe-kvm/`](macos-tahoe-kvm), [`README.md`](README.md) |
| Hardware claims and benchmarks | [`configs/MacPro6,1/README.md`](configs/MacPro6,1/README.md), [`benchmarks/baseline-6.19-stock.txt`](benchmarks/baseline-6.19-stock.txt) | [`README.md`](README.md), [`docs/gpu-acceleration.md`](docs/gpu-acceleration.md) |

## Validation

- Run `bash -n` on every changed shell script.
- Run `python3 -m py_compile scripts/audit-modules.py` after Python changes.
- For docs-only changes, verify the relative links from the edited file, especially links to nested READMEs and docs.
- For Arch packaging changes on an Arch-like host, run `cd packaging/arch && makepkg --nobuild`.
- For raw builder changes on a Linux build host, run `./scripts/build.sh MacPro6,1 <kernel-version>`.
- If a heavy build, ISO build, or hardware boot test was not run, say so explicitly in the change summary.

## Watchpoints

- [`benchmarks/baseline-6.19-stock.txt`](benchmarks/baseline-6.19-stock.txt) is captured output, not normalized metadata. Verify the platform label before quoting it as a canonical hardware statement.
- [`configs/MacPro6,1/config-stock-6.19`](configs/MacPro6,1/config-stock-6.19) is a historical stock 6.19 baseline, not a same-version mirror of the current package build path. Treat its diffs as directional, not as a perfect apples-to-apples comparison with `7.0rc1` packaging inputs.
- The repo says "no initramfs required", but [`packaging/arch/linux-macpro61.install`](packaging/arch/linux-macpro61.install) still copies an initramfs to the ESP if one exists. Treat initramfs support as tolerated fallback, not the primary boot design.
- The KVM docs describe a `BaseSystem.img` flow in some places, while launcher scripts may still reference `.dmg` or generated recovery files. Keep the recovery image format consistent when refactoring.
- [`configs/MacPro6,1/patches/`](configs/MacPro6,1/patches) is currently a placeholder for model-specific patches. Do not confuse it with the active Arch/Cachy patch stack.

Use [`MAP.md`](MAP.md) when you need the per-file purpose, directory relationships, or a starting point for a specific task.
