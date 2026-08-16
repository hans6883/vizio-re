# Attack surface

The device exposes attack surface across seven categories: network services,
UPnP/DLNA control-point behavior, the widget-engine JavaScript runtime, ACR
beaconing, DRM subsystems, boot-chain weaknesses, and physical memory access.

## Network services

### Active listeners (default)

| Port | Bind | Process | Purpose | Risk |
|------|------|---------|---------|------|
| 8963/tcp | 127.0.0.1 | SkypeKit 3.5.1 | Discontinued Skype IPC | Low (loopback) |
| 13000/tcp | 0.0.0.0 | `dtv_svc` | NFLC UPnP event sink | **High** (confirmed DoS) |
| 1900/udp | 0.0.0.0 | `dtv_svc` | SSDP multicast | High (discovery entry point) |
| 51456/udp, 59000/udp | 0.0.0.0 | `dtv_svc` | Undocumented DTV services | Medium |

### Dormant but compiled-in

The firmware `/etc/inetd.conf` defines **FTP (21), telnet (23), rsh (514), and
rlogin (513), all as root with no authentication**. `inetd` is not running by
default, but there is no firewall on the device (iptables is not compiled into
the kernel). Any code path that starts `inetd` with the shipped config instantly
exposes four unauthenticated root services on the LAN.

## Inter-process communication

### Named FIFOs

`dtv_svc` and the application manager communicate over world-accessible FIFOs in
`/tmp` (`fifo_am_read`, `fifo_am_write`, `fifo_dtv_write`). No authentication is
performed on FIFO contents, so any local process can drive application
lifecycle (start/stop/restart) by writing to them.

### DirectFB Fusion shared memory

The widget engine and `dtv_svc` share framebuffer memory through DirectFB Fusion
segments in `/tmp` with no access controls (~100 MB across several segments). A
process with access can inject display content (phishing overlays), observe
input events, or corrupt rendering state.

## Physical memory: `/dev/mem`

`dtv_svc` holds `/dev/mem` open, and the kernel does not restrict `/dev/mem`
access on this build. Any process can therefore map and read/write **arbitrary
physical memory** — hardware registers, DMA buffers, and kernel memory included.
From a post-compromise standpoint this is the single most dangerous property of
the platform: it collapses any code-execution foothold into full physical
control of the SoC.

## Filesystem

| Path | Property | Consequence |
|------|----------|-------------|
| `/3rd_rw/` | Writable, UBIFS, survives factory reset | Primary persistence target |
| `/3rd_rw/lib/` | On `LD_LIBRARY_PATH` before apps start | Library-preload hijack (see [persistence](05-persistence.md)) |
| `/3rd_rw/yahoo_widget/data/Overlay/Script/` | World-writable, loaded at boot | JS overlay injection (see [widget engine](03-widget-engine-security.md)) |
| DRM keybox path | World-writable (`-rw-rw-rw-`) | DRM identity readable/replaceable by any process |

The recurring problem is not any single file but the combination: writable
locations that are read by root-privileged, integrity-protected code at boot,
none of which is re-verified or reset.

## Hardware interfaces

USB mass storage mounts automatically (FAT32) and is a viable payload-staging
vector. HDMI-CEC, IR remote, and multiple tuner threads are present but were not
a focus of this analysis.

## Summary of the surface

The device was manufactured with **no authentication on anything**, no process
isolation, no firewall, an open `/dev/mem`, and writable code paths that outlive
a factory reset. Individually each is a serious weakness; together they mean that
any foothold — local or, via the widget engine, remote — escalates trivially to
persistent root.
