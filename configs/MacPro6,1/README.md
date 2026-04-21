# Mac Pro 6,1 (Late 2013) — "Trash Can"

## Hardware Specifications

| Component | Details | Kernel Driver | Config Option |
|-----------|---------|--------------|---------------|
| **CPU** | Intel Xeon E5-1620 v2 to E5-2697 v2 (Ivy Bridge-EP) | — | CachyOS base + `-march=ivybridge -O3` via KCFLAGS |
| **GPU** | 2x AMD FirePro D300/D500/D700 (Pitcairn or Tahiti, GCN 1.0 / Southern Islands, PCI `1002:6819` or `1002:6798`) | `amdgpu` | Raw model config: `CONFIG_DRM_AMDGPU=m`, `CONFIG_DRM_AMDGPU_SI=y`; Arch package path forces `CONFIG_DRM_AMDGPU=y` |
| **Ethernet** | Broadcom BCM57762 Dual Gigabit | `tg3` | `CONFIG_TIGON3=y` |
| **Wi-Fi** | Broadcom BCM4360 802.11ac | `wl` (proprietary) | Out-of-tree: `broadcom-wl-dkms` (AUR) |
| **Audio** | Intel HDA + Cirrus Logic CS4206 | `snd_hda_intel` | `CONFIG_SND_HDA_INTEL=y`, `CONFIG_SND_HDA_CODEC_CIRRUS=y` |
| **Storage** | Apple PCIe SSD (AHCI) + aftermarket NVMe | `ahci`, `nvme` | `CONFIG_SATA_AHCI=y`, `CONFIG_BLK_DEV_NVME=y` |
| **Thunderbolt** | Intel DSL5520 Falcon Ridge (Thunderbolt 2) + Light Ridge | `thunderbolt` | `CONFIG_USB4=y` |
| **USB 3.0** | Fresco Logic FL1100 | `xhci_hcd` | `CONFIG_USB_XHCI_HCD=y` |
| **USB 2.0** | Intel C600 EHCI + Pericom (via TB) | `ehci-hcd`, `ohci-hcd` | `CONFIG_USB_EHCI_HCD=y` |
| **Thermal** | Apple SMC | `applesmc` | `CONFIG_SENSORS_APPLESMC=y` |
| **NVMe** | Aftermarket via PCIe adapter (common upgrade) | `nvme` | `CONFIG_BLK_DEV_NVME=y` |
| **Bluetooth** | Broadcom (via USB) | `btusb` | `CONFIG_BT_HCIBTUSB=y` |
| **Boot** | EFI | — | `CONFIG_EFI=y`, `CONFIG_EFI_STUB=y` |

## Performance Patches

Applied in the Arch packaging path (`packaging/arch/PKGBUILD`), not by the raw `scripts/build.sh` path:
- **BORE** scheduler (Burst-Oriented Response Enhancer)
- **BBR3** TCP congestion control
- CachyOS kernel tweaks and fixes
- HDMI improvements

## GPU Details

The D700 is based on AMD's Tahiti XT GPU (same silicon as the Radeon HD 7970). It's GCN 1.0 (Southern Islands). Despite Apple marketing referencing "FirePro D700", `lspci` identifies them as `1002:6798` and the amdgpu driver initializes them as TAHITI.

- **Kernel driver:** the trimmed raw model config keeps `amdgpu` as a module (`CONFIG_DRM_AMDGPU=m`) with SI support enabled and disables legacy `radeon`; the Arch packaging path forces `amdgpu` built-in during `prepare()` but still starts from a more generic package seed config
- **Firmware:** Tahiti: `tahiti_{ce,mc,me,pfp,rlc,smc}.bin` — Pitcairn: `pitcairn_{ce,mc,me,pfp,rlc,smc}.bin`
- **Mesa driver:** `radeonsi` (OpenGL), `RADV` (Vulkan)
- **Kernel 7.0:** Mature amdgpu SI support

## Hardware Compatibility Matrix

| Feature | Status | Notes |
|---------|--------|-------|
| Display (DP) | ✅ Works | Via amdgpu SI legacy display path (`amdgpu.dc=0`) |
| Display (HDMI) | ✅ Works | Via amdgpu SI legacy display path (`amdgpu.dc=0`) |
| Display (TB) | ⚠️ Works with issues | Log spam — see Known Issues |
| OpenGL | ✅ Works | Via Mesa radeonsi |
| Vulkan | ✅ Works | Via Mesa RADV |
| GPU Compute | ❌ No ROCm | ROCm does not support Southern Islands; OpenCL via Mesa rusticl only |
| Ethernet | ✅ Works | Both ports via tg3 |
| Wi-Fi | ⚠️ Requires proprietary driver | Install `broadcom-wl-dkms` (AUR) + `linux-macpro61-headers` for DKMS |
| Audio (3.5mm) | ✅ Works | Via snd_hda_intel + Cirrus codec |
| Audio (HDMI) | ✅ Works | Via amdgpu HDMI audio |
| Thunderbolt | ⚠️ Works with issues | Hotplug log spam |
| USB 3.0 | ✅ Works | Via xHCI |
| Sleep/Wake | ❌ Disabled | Explicitly disabled in kernel config |
| NVMe + TRIM | ✅ Works | Built-in; enable `fstrim.timer` or `discard` mount option |
| Bluetooth | ✅ Works | Via btusb |
| Fan Control | ⚠️ Requires setup | Via applesmc; install `macfanctld` (AUR) for automatic control |
| Temperature Sensors | ✅ Works | Via applesmc + hwmon |

## Known Issues

### Thunderbolt Display Log Spam
The 6,1 generates excessive kernel log messages related to Thunderbolt hotplug detection and ACPI errors with Thunderbolt displays connected. Mitigations:

- `CONFIG_DYNAMIC_DEBUG=y` — selectively silence at runtime
- `acpi_osi=Darwin` boot parameter — may calm firmware re-enumeration
- DSDT override — suppress spurious ACPI events
- `kernel.printk = 3 4 1 3` — reduce console noise
- Patches in `patches/` directory (when available)

### GPU Ring Timeouts Under Sustained Load
Running heavy GPU compute (e.g., LLM inference) on the FirePro D300/D500/D700 can cause ring timeout errors and system crashes. The packaged kernel path forces built-in amdgpu and embeds the relevant firmware; the raw model config keeps `amdgpu` modular, so stability comparisons should account for which build path you tested.

### Sleep/Wake
Explicitly disabled in the kernel config (`CONFIG_SUSPEND`, `CONFIG_ACPI_SLEEP`, `CONFIG_HIBERNATION` all unset). Sleep/wake has always been unreliable on the Mac Pro 6,1 under Linux, and this is a workstation/server kernel — if it's on, it's on.

## Model Variants

| Model | CPU | GPU | RAM |
|-------|-----|-----|-----|
| Base | E5-1620 v2 (4C/8T, 3.7GHz) | 2x D300 (2GB) | 16GB |
| Mid | E5-1650 v2 (6C/12T, 3.5GHz) | 2x D500 (3GB) | 16GB |
| High | E5-1680 v2 (8C/16T, 3.0GHz) | 2x D700 (6GB) | 32/64GB |
| BTO Max | E5-2697 v2 (12C/24T, 2.7GHz) | 2x D700 (6GB) | 64GB |

The kernel config includes firmware for all GPU variants: Tahiti (D500/D700) and Pitcairn (D300). No changes needed regardless of which model you have.

All Mac Pro 6,1 configurations share the same Thunderbolt controller, ethernet, Wi-Fi, and audio. The only hardware differences are CPU core count (4-12 cores, all Ivy Bridge-EP), RAM amount (no kernel impact), and GPU variant (firmware handled above). Aftermarket NVMe via PCIe adapter is extremely common — the kernel config includes both NVMe (`CONFIG_BLK_DEV_NVME=y`) and AHCI (`CONFIG_SATA_AHCI=y`) built-in.

The repository currently carries two kernel build paths with different starting points:

- [`config`](config) is the trimmed raw model config used by [`../../scripts/build.sh`](../../scripts/build.sh)
- [`../../packaging/arch/config`](../../packaging/arch/config) is the package seed config used by [`../../packaging/arch/PKGBUILD`](../../packaging/arch/PKGBUILD), which then forces critical options to `=y`

## PCI Device IDs

```
GPU 1:      1002:6798  AMD Tahiti XT [FirePro D700]
GPU 2:      1002:6798  AMD Tahiti XT [FirePro D700]
HDMI Audio: 1002:aaa0  AMD Tahiti HDMI Audio (x2)
HDA:        8086:1d20  Intel C600/X79 HD Audio (Cirrus Logic CS4206)
NIC 1:      14e4:1682  Broadcom BCM57762 Gigabit Ethernet
NIC 2:      14e4:1682  Broadcom BCM57762 Gigabit Ethernet
NIC 3:      14e4:16b0  Broadcom BCM57761 (via Thunderbolt)
Wi-Fi:      14e4:43a0  Broadcom BCM4360 802.11ac
TB (FR):    8086:156c  Intel DSL5520 Falcon Ridge NHI (Thunderbolt 2)
TB (LR):    8086:1513  Intel CV82524 Light Ridge (Thunderbolt 1)
USB 3.0:    1b73:1100  Fresco Logic FL1100 xHCI
USB 2.0:    8086:1d26  Intel C600/X79 EHCI
USB (TB):   12d8:400e  Pericom OHCI/EHCI (via Thunderbolt)
PCIe SW:    10b5:8723  PLX PEX 8723 6-port PCIe Switch
SSD:        144d:a801  Samsung Apple-slot AHCI (varies)
SMBus:      8086:1d22  Intel C600/X79 SMBus
MEI:        8086:1d3a  Intel Management Engine Interface
```

Use `lspci -nn` on your system to verify.
