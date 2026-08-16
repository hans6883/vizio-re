# Widget engine security model

The E320i-A0 runs the **Yahoo Konfabulator** widget engine (a.k.a. Yahoo
Connected TV / Yahoo TV Widgets). It is the richest attack surface on the device
because it combines a JavaScript runtime with a native bridge to the shell — all
running as root, with no sandbox.

## Components

| Component | Role |
|-----------|------|
| `Konfabulator` | Entry point (~81 KB) |
| `libKonfabulator.so` | Core: JS runtime, XML parser, HTTPS client, filesystem API, shell bridge (~17 MB) |
| `libPluginTv.so` / `libPluginTvOEM.so` | TV platform / OEM API plugins |
| SpiderMonkey | Embedded Mozilla JS engine |

The engine runs as a single root process. All widgets share one process, one JS
context, and root privileges.

## The JavaScript → shell bridge

The core library exports functions that bridge widget JavaScript directly to
native execution:

- `RunCommand` / `RunCommandInBg` — execute an arbitrary shell command (the
  latter in the background). Commands run as root.
- `OemLaunchApplication` — launch OEM apps from widget context.
- A complete `Filesystem` API — `Open`, `WriteFile`, `Copy`, `Move`, `Remove`,
  `CreateDirectory`, `GetDirectoryContents`, `Zip`, `Unzip`, `GetMD5`.

In other words, any JavaScript the engine loads is as powerful as a root shell.
There is no capability gate between "widget script" and "system command."

## The "sandbox" is not a sandbox

`libKonfabulator` exposes a `GetSandboxDir`, and the directory it names
(`.../data/sandbox/`) holds compiled JS bytecode (`.js.1.o`). Analysis confirms
this is a **compilation cache**, not a security boundary. There is no process
isolation, no capability restriction, and no filesystem/network confinement.
Every widget runs in the same root process with full access to the shell bridge.

## Signing policy: wildcard on the dangerous categories

Widget packages are signed, and configuration restricts *some* categories to
specific signer fingerprints. But two configuration files interact badly:

- **`config-oem.xml`** restricts gallery/super/oem/login/widget categories to
  specific fingerprints — **but leaves every `tvapi.*` category set to `*`
  (any signer)**.
- **`config.xml`** (the base config) goes further: it sets *all* API category
  policies to `*` **and** leaves the installer/package required-fingerprint
  fields empty.

Net effect: regardless of signing enforcement elsewhere, **any widget can reach
the `tvapi.*` system-control, volume-control, and tuner APIs**, because the
wildcard is present in both configs. If the OEM config ever fails to parse and
the engine falls back to base config, *all* widgets become installable unsigned.
The signing scheme also relies on SHA-1 fingerprints, deprecated since 2017.

(The specific signer fingerprints and the encrypted "super-widget" blobs are
device-firmware artifacts and are intentionally omitted from this repository.)

## Script-overlay injection (the critical one)

The base config enables a **script overlay** feature:

```
script-dir-overlay-enabled = true
script-dir-overlay-sources  = Bootstrap;Framework;Platform;Images;platform.js
```

When enabled, the engine checks a writable overlay directory **first**, before
loading the read-only framework scripts from `/3rd/`. The overlay directory
lives on the writable, reset-surviving `/3rd_rw` partition and is **world-writable
(`-rw-rw-rw-`)** with **no signing and no integrity check**.

So a JavaScript file placed at the right overlay path (e.g. the framework's
common loader) **overrides** the genuine read-only script, and is loaded by the
root widget process on every boot. Because that script can call `RunCommand`,
this is a complete, code-only persistence and root-execution path:

```
1. Write malicious JS into the overlay dir (Framework loader path)
2. Reboot (or wait for the engine to restart)
3. Engine loads the overlay copy instead of the read-only original
4. The overlay JS calls RunCommand(...) → arbitrary commands as root
```

No ELF work, no library compilation, no signing bypass — just a writable file
that verity-protected root code trusts. It also survives factory reset, because
the overlay directory is on `/3rd_rw`.

## Remote angle: dead gallery + DNS

The engine is configured with `bootstrap.contact-on-startup = 1` and fetches its
widget gallery over **HTTP** (base config) from Yahoo endpoints that Yahoo has
long since discontinued. On a network where DNS is controlled, those dead
endpoints can be answered, and there is no certificate pinning on the bootstrap
path — turning "abandoned cloud dependency" into a MITM opportunity to serve
widget content to the engine.
