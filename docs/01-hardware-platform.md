# Hardware and platform

## Device

| Property | Value |
|----------|-------|
| Model | Vizio E320i-A0 (32" LCD smart TV, 2013) |
| Platform | VIZIO VIA (Vizio Internet Apps), model class MG111M1.1 |
| SoC | MediaTek MT5369 (ARM Cortex-A9, ARMv7 little-endian) |
| Kernel | Linux 2.6.35 `#1 PREEMPT`, built 2015 |
| C library | uClibc |
| TLS/crypto | OpenSSL 1.0.0 (EOL Dec 2015), TLS 1.0 only |
| Widget engine | Yahoo Konfabulator / Yahoo Connected TV |
| JS runtime | SpiderMonkey (embedded in the widget engine) |
| Display | DirectFB framebuffer, 960×540 |

Every major software component predates roughly 2012. OpenSSL 1.0.0 alone
carries a long list of critical CVEs (Heartbleed, POODLE, FREAK, Logjam, …).

## Partition and storage layout

The device uses a dual **A/B** partition scheme on raw NAND, allowing atomic
firmware updates with bootloader rollback.

| Partition | Purpose | Approx size | Integrity |
|-----------|---------|-------------|-----------|
| uboot A/B | Bootloader (U-Boot) | 20 MB each | — |
| kernel A/B | Linux 2.6.35 | 20 MB each | — |
| rootfs A/B | Root filesystem | 20 MB each | dm-verity |
| basic A/B | Core apps + libraries | 200 MB each | dm-verity |
| sig A/B | Update signatures | — | — |
| channel A/B | Tuner presets | — | — |
| 3rd | Third-party apps | 400 MB | squashfs (likely dm-verity) |
| **3rd_rw** | **Writable app data** | **500 MB** | **UBIFS — no integrity protection** |
| perm | Factory identity | 80 MB | UBIFS |

### The security-relevant asymmetry

`dm-verity` cryptographically protects `rootfs` and `basic`, so the kernel, the
master TV service, and the core libraries cannot be modified without breaking
the hash chain. **`/3rd_rw` is explicitly not covered by dm-verity.** It is
writable, it is on the library search path, and — critically — there is no
evidence in the boot chain of any routine that wipes it during a factory reset.

That single design choice turns `/3rd_rw` into a clean persistence surface: code
planted there is loaded by verity-protected binaries at boot, yet is itself
outside the integrity boundary and outside the reset path. This theme recurs
throughout the findings.

## Runtime process model

Two applications are auto-started, in order:

1. **`dtv_svc`** (`/basic/dtv_svc`, ~2.9 MB) — the master TV service. Handles
   tuner control, framebuffer rendering, UPnP/DLNA, and application management.
   It runs as **root** and holds an open descriptor to **`/dev/mem`**.
2. **Konfabulator** (`/3rd/yahoo_widget/Konfabulator`) — the widget engine, also
   **root**, ~400 MB virtual footprint. Hosts all widget JavaScript in a single
   process with no isolation.

There is exactly one account on the system — `root`, with no password — and no
privilege separation between components. Every process of interest runs as uid 0.
