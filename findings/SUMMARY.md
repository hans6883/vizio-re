# Findings summary

Severity reflects impact on a device of this class (2013, end-of-life, no vendor
updates) on a shared local network. "Local-UI" means physical/UI access to the
TV; "LAN" means any device on the same network with no authentication.

| # | Finding | Vector | Access | Severity |
|---|---------|--------|--------|----------|
| 1 | OS command injection via WiFi "Hidden Network" SSID field → root | SSID → shell, unsanitized | Local-UI | Critical |
| 2 | Widget script-overlay injection: world-writable, unsigned JS loaded by root at boot; survives factory reset | `/3rd_rw` overlay dir | Local (any process) / remote via #7 | Critical |
| 3 | JS→shell bridge (`RunCommand`) with no sandbox; all widgets run as root in one process | Widget engine design | Post-compromise / via #2, #7 | Critical |
| 4 | Library-preload persistence on `/3rd_rw` (on loader path, not verity-covered, not reset-wiped) | `LD_LIBRARY_PATH` hijack | Post-compromise | Critical |
| 5 | `/dev/mem` open and unrestricted → arbitrary physical memory access | Kernel/`dtv_svc` | Post-compromise | Critical (escalation) |
| 6 | Remote UPnP DoS via `Content-Length: -1` on the 13000/tcp event sink; power cycle to recover | `dtv_svc` UPnP | LAN, unauth | High |
| 7 | Dead widget gallery fetched over HTTP with no cert pinning → widget MITM where DNS is controlled | Konfabulator bootstrap | LAN + DNS control | High |
| 8 | `tvapi.*` API categories accept any widget signer (`*`); base config has empty signing requirements | Widget signing policy | Via widget load | High |
| 9 | No authentication anywhere: root has no password; `inetd` ships FTP/telnet/rsh/rlogin as root | System config | LAN if `inetd` started | High (latent) |
| 10 | World-writable DRM keybox and plaintext device DRM/identity material | Filesystem perms | Local (any process) | Medium–High |
| 11 | Always-on ACR fingerprint beaconing (~5.5 s) over TLS 1.0; config channel can push overlay alerts | TVIS ACR client | Default behavior | Medium (privacy) |
| 12 | Obsolete crypto/software stack (OpenSSL 1.0.0, TLS 1.0, kernel 2.6.35, `libxml2` 2.7.7) | Platform-wide | — | Medium (systemic) |

## The through-line

No single finding is surprising for the era; the compounding is what matters.
Integrity protection (dm-verity) on the root filesystem is neutralized because
the protected, root-privileged binaries load code and config from writable
locations that are excluded from both verity **and** factory reset. Combined with
zero authentication, no process isolation, and an open `/dev/mem`, any foothold —
local via the SSID field, or remote via the widget/DNS path — becomes persistent
root that a user cannot clear without reflashing.

## Not fully explored

Honest gaps, so the summary isn't read as exhaustive:

- Much of the UPnP event-sink fuzz corpus (SID/header overflows, SEQ edges, deep
  nesting, format strings in properties, large bodies) was **never reached**
  because the `Content-Length: -1` wedge cut runs short — "not tested," not
  "safe."
- The master service (`dtv_svc`) DLNA/XML parser was not reversed to memory-
  corruption depth; it is the strongest candidate for remote code execution and
  remains open work.
- The encrypted "super-widget" configuration blobs were not decrypted.
