# linux-mac

Custom Linux kernel for the Mac Pro 6,1 (Late 2013). CachyOS-based with BORE scheduler; the packaged Arch path forces critical drivers built-in, embeds GPU firmware, and boots to desktop with no initramfs required.

> Original upstream note: `wolffcatskyy/linux-mac` and `wolffcatskyy/cachyos-macpro-iso` were archived on March 10, 2026. Treat this repository (`ishad0w/linux-trash-can`) and the docs in this tree as the maintained source of truth.

## What This Is

A kernel config set and Arch PKGBUILD for a Mac Pro 6,1 kernel line centered on the `7.0rc1` package path. CachyOS 7.0 base with BORE scheduler and BBR3; the package build forces critical Mac Pro drivers built-in and embeds GPU firmware in the kernel.

If you want the full CachyOS/BORE/BBR3 patch stack, use [`packaging/arch/`](packaging/arch). [`scripts/build.sh`](scripts/build.sh) builds the raw model config against vanilla kernel.org sources and only applies model-local patches; it is a separate raw-builder path, not the `7.0rc1` Arch package path.

- **All GPU variants** — D300 (Pitcairn), D500 (Tahiti), D700 (Tahiti XT), firmware baked in
- **CachyOS performance** — BORE scheduler, BBR3 congestion control, `-march=ivybridge -O3`
- **KVM built-in** — run macOS Tahoe in QEMU
- **NVMe + TRIM** — aftermarket NVMe drives work out of the box

## Hardware Support

| Feature | Status | Notes |
|---------|--------|-------|
| GPU (D300/D500/D700) | Working | Packaged kernel forces amdgpu built-in; radeonsi/RADV via Mesa |
| Display (DP/HDMI) | Working | Via amdgpu SI legacy display path (`amdgpu.dc=0`) |
| Vulkan / OpenGL | Working | Mesa RADV / radeonsi |
| GPU Compute | Limited | OpenCL via rusticl only — no ROCm for Southern Islands |
| Ethernet | Working | Both ports via tg3 + Broadcom PHY |
| Wi-Fi | Proprietary | `broadcom-wl-dkms` (AUR) + headers package |
| Audio | Working | Intel HDA + Cirrus CS4206, HDMI/DP via amdgpu |
| USB 3.0 | Working | xHCI |
| Thunderbolt 2 | Partial | Works with log spam |
| NVMe + TRIM | Working | Built-in; enable `fstrim.timer` |
| Bluetooth | Working | Broadcom via btusb |
| KVM | Working | macOS Tahoe virtualization |
| Fans / Thermal | Working | applesmc + hwmon; install `macfanctld` (AUR) |
| Sleep/Wake | Disabled | Unreliable on this hardware |

## Quick Start

```bash
git clone https://github.com/ishad0w/linux-trash-can.git
cd linux-trash-can/packaging/arch
makepkg -s
sudo pacman -U linux-macpro61-*.pkg.tar.zst
sudo poweroff  # Apple EFI needs cold boot — never reboot when switching kernels
```

## Important

**Always power off (not reboot) when switching kernels.** Apple EFI needs a cold boot to reinitialize the GPU.

## CachyOS Patches

Built on the CachyOS 7.0 patch set:
- **BORE** — Burst-Oriented Response Enhancer scheduler
- **BBR3** — Google TCP congestion control v3
- **CachyOS tweaks** — kernel optimizations
- **HDMI improvements** — display fixes

## Documentation

- [Repository Map](MAP.md) -- file-by-file layout, change surfaces, and entry points
- [Working Guide](AGENTS.md) -- contributor and automation rules, repo invariants, sync rules
- [Technical Debt](TECH-DEBT.md) -- confirmed repo-level debt, stale paths, and cleanup priorities
- [Image Workflow](image/README.md) -- current in-tree ISO/image preparation path
- [GPU Acceleration Guide](docs/gpu-acceleration.md) -- full stack explainer, what works, performance tuning, roadmap
- [Mesa Setup](docs/mesa.md) -- driver config, environment variables, multi-GPU
- [macOS Tahoe KVM](docs/kvm-macos.md) -- run macOS in a VM on this kernel
- [PVG Roadmap](docs/pvg-linux.md) -- GPU acceleration for macOS VMs
- [Archived CachyOS ISO Repo](https://github.com/wolffcatskyy/cachyos-macpro-iso) -- historical reference only; archived on March 10, 2026

## Roadmap

| Status | Milestone |
|--------|-----------|
| Done | CachyOS 7.0 base with BORE, BBR3, built-in amdgpu |
| Done | All GPU variants, verified against lspci |
| Done | KVM + macOS Tahoe virtualization |
| Historical | Archived CachyOS-based Mac Pro ISO repo exists as reference only; current image work lives under [`image/`](image/) |
| Done | Driver trimming — removed ~3000 unused config options for faster builds |


## Boot Configuration

**systemd-boot** with ESP at \`/boot/efi/\` (FAT32).

### Gotchas

1. **ESP vs /boot** - pacman installs to \`/boot/\` (root partition) but systemd-boot reads from \`/boot/efi/\` (ESP). The package install hook syncs automatically.

2. **Cold boot only** - Apple EFI needs full power cycle for GPU init. The package masks \`reboot.target\` and aliases \`reboot\` to \`poweroff\` automatically.

3. **Boot entries** - Default: \`linux-macpro61.conf\` (custom kernel), fallback: your distro's stock entry (for example \`arch-6.19.conf\` on the original Arch test system).

## License

GPL-2.0 (same as the Linux kernel)
