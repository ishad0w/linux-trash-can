# linux-macpro61 Patch Inventory

Current package base: CachyOS `cachyos-7.1.3-1` release tarball via `PKGBUILD`.

The files `0001-bore.patch` through `0005-hdmi.patch` are retained as carry-patch snapshots from the earlier 7.0rc1 packaging path. They are not disposable: this project exists because Mac Pro 6,1 hardware needs behavior that generic Linux/CachyOS does not always provide out of the box.

## Active Patch Sources

Applied by default in `PKGBUILD`:

- CachyOS release tarball: `https://github.com/CachyOS/linux/releases/download/cachyos-7.1.3-1/cachyos-7.1.3-1.tar.gz`
- `dkms-clang.patch` from `cachyos/kernel-patches/master/7.1`
- `0001-bore-cachy.patch` from `cachyos/kernel-patches/master/7.1`
- Mac Pro 6,1 config enforcement in `prepare()`: embedded D300/D500/D700 firmware, built-in `amdgpu`, `tg3`, Broadcom PHY, Apple SMC, AHCI/NVMe, USB4, HDA, KVM, OverlayFS, and BBR default.

## Retained Local Snapshots

| File | Current 7.1.3 status | Notes |
|------|------|------|
| `0001-bore.patch` | Superseded by upstream `0001-bore-cachy.patch` | Dry-run against prepared 7.1.3 reports mostly reversed/already-applied hunks. |
| `0002-bbr3.patch` | Mostly superseded by the 7.1.3 source/config | `CONFIG_TCP_CONG_BBR3` and BBR3 TCP code are already present. Re-enable only after a clean 7.1.3 port. |
| `0003-cachy.patch` | Mostly superseded by the CachyOS 7.1.3 release tarball | Verified examples include `amdgpu_ignore_min_pcap` and AMD private color/KMS changes already present. |
| `0004-fixes.patch` | Needs port audit before enabling | Some hunks are already upstream; others fail on 7.1.3 scheduler/core/audio files and must not be applied with `|| true`. |
| `0005-hdmi.patch` | Mostly present, but still needs port audit | 7.1.3 already contains HDMI VRR/ALLM/PCON symbols such as `ADAPTIVE_SYNC_TYPE_HDMI`, `hdmi_allm_capable`, `drm_hdmi_vrr_cap`, and `DC_OVERRIDE_PCON_VRR_ID_CHECK`; failed hunks need review before claiming full parity. |

## Rules

- Do not delete local patch snapshots unless the replacement or upstream supersession is documented here.
- Do not apply carry patches with `patch ... || true`; partial application is worse than a hard failure.
- If a retained patch is re-enabled, add it to `source=()` and `b2sums=()`, apply it through the strict patch loop in `prepare()`, and run at least `makepkg --nobuild --nodeps --skippgpcheck`.
- Hardware-sensitive display, firmware, Apple EFI, Thunderbolt, and Broadcom networking changes should be treated as Mac Pro carry-layer work even when their original patch came from CachyOS.
