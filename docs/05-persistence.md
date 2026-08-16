# Persistence: library preload on a reset-surviving partition

Once root is obtained, the platform offers an unusually durable persistence
surface. This documents the technique conceptually; it does not ship a payload.

## The primitive

`/3rd_rw/lib/` is on the dynamic-linker search path (`LD_LIBRARY_PATH`) that the
boot scripts set **before** any application starts, and `/3rd_rw` is:

- **writable**,
- **not** covered by dm-verity, and
- **not** wiped by factory reset.

A shared library placed there can therefore be loaded ahead of the genuine
system copy, by a root-privileged process, on every boot — and neither a reboot
nor a factory reset removes it.

## Why the WiFi scan is a convenient trigger

The WiFi stack loads a small wireless-info library on each scan. That library is
resolved through the search path that includes `/3rd_rw/lib/`, and WiFi scans
happen periodically on their own. A preload library placed at that name is thus
executed repeatedly with no user interaction, purely as a side effect of normal
device operation.

## Constructor-based launch

The technique uses an ELF **constructor** (`.init_array`) so that merely loading
the library runs code — no exported symbol needs to be called. Practical
considerations observed on this platform:

- The library must not depend on a full libc; a minimal, raw-syscall stub avoids
  link-time and load-time coupling to the exact system libc.
- The constructor should **double-fork** and detach so it does not block or crash
  the host process (the WiFi machinery, which must keep working).
- A guard flag (a well-known temp file) prevents relaunching a new background
  listener on every scan, which would otherwise pile up processes.

The result is a root listener that comes back after reboots and factory resets.
A second, independent persistence path exists entirely in JavaScript via the
widget **script overlay** (see
[widget engine](03-widget-engine-security.md)) — no native code required.

## Why this is hard to remove

Because the payload lives outside the dm-verity boundary and outside the reset
path, the normal user remedies do not clear it:

- A reboot re-runs it.
- A factory reset does not touch `/3rd_rw`.
- Only a full firmware reflash of the writable partition would remove it.

## Defensive takeaway

The lesson generalizes beyond this device: a writable directory that is on the
loader search path of a privileged, integrity-protected process — and that is
excluded from factory reset — is a persistence gift. Integrity protection on the
root filesystem is undermined if the trusted binaries there load code from an
untrusted, resettable-but-not-reset partition. Reset routines must cover every
writable location that feeds privileged startup.
