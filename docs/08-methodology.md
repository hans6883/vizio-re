# Methodology

How the analysis was carried out, for readers who want to reproduce the *approach*
on hardware they own.

## Access and acquisition

- Initial code execution came from the local SSID command-injection path
  (see [04-ssid-command-injection.md](04-ssid-command-injection.md)), which
  yields a root shell.
- With root, the writable `/3rd_rw` partition and the read-only `/3rd` and
  `/basic` trees were enumerated: process list, open file descriptors, listening
  sockets, mounts, `inetd`/service configuration, boot scripts, and the widget
  engine's configuration and framework.
- The device's BusyBox is minimal (no `wget`/`nc`/`tftp`/`httpd`), so file
  transfer off the device was done over the shell channel and via the
  auto-mounting USB path.

## Static binary analysis

- The widget engine core and several platform libraries were examined as ARM32
  ELF: exports, linked libraries, and string/symbol inspection identified the
  JS→shell bridge, the filesystem API, the trust/`AllowLogin` functions, and the
  config accessors.
- Security properties were noted per binary (stripped, PIC, stack canaries).
  Several small libraries were built without stack protection.

> Note: no vendor binaries are redistributed in this repo. The analysis is
> described; reproducing it requires dumping the binaries from your own device.

## Dynamic testing

- **UPnP/DLNA:** a fake MediaServer answered the TV's `M-SEARCH`, then served
  malformed device-description and DIDL-Lite XML from a rotating corpus, watching
  for crashes and for the event-sink wedge. Discovery had to be re-triggered
  frequently because the TV ignores already-known servers (UUID rotation was used
  to force rediscovery).
- **Network config channels:** the TV's DNS was pointed at a controlled resolver
  in the lab so that outbound TV/widget/ACR endpoints could be observed and, where
  relevant, answered. A permissive TLS 1.0 endpoint was needed because modern
  TLS stacks refuse to speak TLS 1.0 with the device.
- **Denial-of-service isolation:** `Content-Length` edge cases were sent with the
  socket closed immediately after the request, to prove the fault persisted in
  the server state rather than in a held connection.

## Environment hygiene

All testing was done on a personally owned device on an isolated lab network. No
device identifiers, keys, certificates, or network addresses from that
environment are included in this repository; the [findings](../findings/SUMMARY.md)
and docs describe *classes* and *mechanisms*, with instance-specific material
removed.

## Reproducing responsibly

If you own one of these TVs and want to follow along: work on an isolated
network, expect the UPnP DoS to require a power cycle, and remember that anything
you plant on `/3rd_rw` survives a factory reset — keep notes on what you change so
you can undo it, or be prepared to reflash.
