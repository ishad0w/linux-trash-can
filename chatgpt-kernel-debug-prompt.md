# ChatGPT Kernel Debug Prompt

Copy-paste this when stuck on custom kernel issues for Mac Pro 6,1:

This template reflects the historical raw-builder / vanilla-`6.19` debug path shown elsewhere in this repo. If you are debugging the packaged Arch path, replace the kernel-source, boot-parameter, and initramfs details with the `packaging/arch/` equivalents.

---

I'm building a custom Linux kernel for a Mac Pro 6,1 (Late 2013, "trash can"). Help me debug boot/display/driver issues.

## Hardware
- CPU: Intel Xeon E5-1650 v2 (Ivy Bridge-EP), 6c/12t
- GPU: 2x AMD FirePro D700 (Tahiti XT, GCN 1.0 / Southern Islands, PCI ID 1002:6798)
- Network: Broadcom BCM57762 (tg3 driver)
- Display: Thunderbolt 2 / Mini DisplayPort only (no HDMI/DVI)
- EFI: Apple 64-bit UEFI 1.10 (NOT standard PC UEFI)
- Boot: systemd-boot, EFI System Partition at /boot/efi/ (example layout)
- Root: /dev/sda2, Arch Linux (example test system)

## Kernel Source
- Historical raw-builder example: vanilla kernel 6.19 from kernel.org (no Arch patches)
- Config based on stock Arch /proc/config.gz with minimal changes

## Working Boot Parameters
Example boot parameters from a historical raw-builder test system:

```
root=/dev/sda2 rw quiet amdgpu.si_support=1 radeon.si_support=0 amdgpu.dc=0 video=efifb:off amdgpu.lockup_timeout=120000,600000,600000,600000 amdgpu.gpu_recovery=1 amdgpu.job_hang_limit=16 pcie_aspm=off amdgpu.dpm=0 acpi_mask_gpe=0x10000 acpi_enforce_resources=lax
```

The packaged `linux-macpro61` path in this repo currently bakes a smaller default cmdline centered on `acpi_mask_gpe=0x16 amdgpu.si_support=1 amdgpu.dc=0`.

## Known Constraints
1. **kexec is BROKEN** on Apple EFI — GPU cannot reinitialize without cold boot. Always do a full poweroff, not a warm reboot, when changing kernels.
2. **If you use modules or an initramfs, versions must match** — vermagic mismatch = no modules load = black screen. Rebuild the initramfs after `make modules_install`. The packaged Arch path in this repo is designed to boot without initramfs, but still tolerates one as fallback.
3. **amdgpu.si_support=1** is required — D700 is Southern Islands (si), NOT Sea Islands (cik).
4. **amdgpu.dc=0** required — Display Core doesn't work with SI GPUs.
5. **Firmware blobs** (tahiti_*.bin) must either be embedded via `CONFIG_EXTRA_FIRMWARE` or be present in the initramfs if you still boot with one.
6. All display outputs go through Thunderbolt 2 — there is no direct HDMI/DVI.

## Current Status
[DESCRIBE YOUR CURRENT ISSUE HERE]

## What I've Tried
[LIST WHAT YOU'VE ALREADY TRIED]
