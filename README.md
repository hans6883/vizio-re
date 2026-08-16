# vizio-re

Reverse-engineering and security analysis of a **Vizio E320i-A0** smart TV — a
2013-era, end-of-life VIA-platform set built on a MediaTek MT5369 SoC running
Linux 2.6.35 and the Yahoo Konfabulator widget engine.

This repository documents the platform architecture and the security weaknesses
of a smart-TV software stack that is no longer supported by the vendor. It is
published as embedded-security research and RE methodology, not as an attack
toolkit.

## Why this platform is interesting

The VIA (Vizio Internet Apps) generation of MediaTek smart TVs is largely
undocumented publicly. These devices shipped in the millions, reached end of
life years ago, and still turn up on home networks. The E320i-A0 is a useful
case study in how consumer TV firmware of that era treated security: no
authentication, a JavaScript widget engine with an unrestricted native
shell-execution bridge, world-writable DRM material, and writable code paths
that survive a factory reset.

## What's here

| Doc | Topic |
|-----|-------|
| [docs/01-hardware-platform.md](docs/01-hardware-platform.md) | SoC, kernel, partition/memory layout, A/B boot scheme |
| [docs/02-attack-surface.md](docs/02-attack-surface.md) | Network services, IPC, filesystem, `/dev/mem`, hardware interfaces |
| [docs/03-widget-engine-security.md](docs/03-widget-engine-security.md) | Konfabulator model, JS→shell bridge, script-overlay injection, signing policy |
| [docs/04-ssid-command-injection.md](docs/04-ssid-command-injection.md) | Initial-access writeup: command injection via the WiFi SSID field |
| [docs/05-persistence.md](docs/05-persistence.md) | Library-preload persistence on a writable, reset-surviving partition |
| [docs/06-dlna-upnp-dos.md](docs/06-dlna-upnp-dos.md) | UPnP event-sink denial of service and DLNA parser fuzzing |
| [docs/07-acr-privacy.md](docs/07-acr-privacy.md) | Automatic Content Recognition beaconing and its privacy profile |
| [docs/08-methodology.md](docs/08-methodology.md) | How the analysis was done; tooling and RE approach |
| [findings/SUMMARY.md](findings/SUMMARY.md) | All findings with severity |

## What's deliberately *not* here

To keep this a research publication rather than a weapon, this repo excludes:

- **Vendor firmware binaries.** No proprietary `.so`/ELF files, kernel modules,
  or firmware images are redistributed. The analysis describes them; it does not
  ship them.
- **Any device identity or key material.** No DRM keyboxes/keys, device
  certificates, provisioning blobs, serial numbers, MAC addresses, or tokens.
- **Turn-key exploit tooling.** Individual commands used during the research are
  shown (with environment-specific values replaced by placeholders), but no
  drop-in exploit scripts, persistence payloads, or automated attack tools are
  included.
- **Personal infrastructure.** No IP addresses, hostnames, or network details
  from the research environment.

## Scope and ethics

- The subject is a **personally owned** device on a **private lab network**.
- The platform is **end-of-life**; the vendor no longer ships firmware updates
  for it, so there is no patch to coordinate and no fleet actively defended by a
  fix. Findings are published in that context.
- Nothing here targets a live service, a third party, or any device the reader
  does not own. Don't apply any of it to hardware you don't own.

## License

- **Documentation** (everything under `docs/` and `findings/`): CC BY 4.0.
- **Any code samples**: MIT (see [LICENSE](LICENSE)).
