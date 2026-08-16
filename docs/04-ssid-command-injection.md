# Initial access: command injection via the WiFi SSID field

This is the flagship finding and the vector used to gain first code execution on
the device. It is written up here in responsible-disclosure style. The affected
platform is end-of-life and no longer receives vendor firmware updates.

## Summary

The VIA UI's **"Hidden Network"** flow, which lets the user type an SSID
manually, passes that SSID string toward a shell without sanitization. An SSID
containing a backtick-wrapped command therefore causes that command to execute —
as **root** — during the connection attempt.

- **Class:** OS command injection (CWE-78) via an untrusted network-configuration
  field.
- **Privilege:** root (uid 0), because the network stack runs as root.
- **Access required:** local UI interaction to enter the hidden-network SSID.
- **Authentication required:** none beyond physical/UI access to the TV.

## Why "Hidden Network" specifically

Passive WiFi scanning does **not** trigger the vulnerable path — scanned SSIDs
from the air are stored by `wpa_supplicant` as data and never reach a shell. The
injection is in the **manual entry** path: when the user opens *Menu → Network →
Hidden Network* and types an SSID, the VIA UI hands that value to a component
that composes a shell command line with it. That distinction matters both for
understanding the bug and for scoping its reachability: it is a local-input bug,
not a remote "broadcast a malicious SSID and own nearby TVs" bug.

## Mechanism

1. The user enters an SSID in the Hidden-Network dialog.
2. The value flows into network/MAC configuration handling (a
   `*_write_mac_config*`-style routine in the network-info library is the
   suspected sink) and, downstream, into a shell invocation.
3. No escaping or allow-listing is applied to the SSID string.
4. Backtick (`` `…` ``) or `$( … )` content in the SSID is evaluated by the
   shell, executing as root.

The SSID field is length-limited (on the order of ~32 bytes total), which
constrains the payload but is more than enough to bootstrap a larger stage — for
example, executing a short command that reads a second stage from attached USB
storage, which the device auto-mounts.

## Impact

Root command execution from a value the product invites the user to type. Once
root is obtained, the platform offers no barrier to persistence (see
[persistence](05-persistence.md)) and no privilege boundary to cross — every
relevant process already runs as root, and `/dev/mem` is open.

## Root cause and remediation (for platforms of this class)

- **Root cause:** building a shell command line by string-concatenating an
  untrusted configuration field.
- **Fix:** never pass user-supplied network fields through a shell. Use
  `exec`-style APIs with an argument vector, or strictly allow-list the SSID
  character set (SSIDs are octet strings, but the *product's* handling should
  reject shell metacharacters). Treat every field the UI collects as untrusted
  input to be validated at the boundary.

## Disclosure note

The subject device is a personally owned, ~2013, end-of-life TV that the vendor
no longer updates; there is no firmware patch to coordinate and no supported
fleet defended by one. The writeup describes the vulnerability class and its
mechanism at the level needed to understand and defend against it, and omits a
turn-key payload.
