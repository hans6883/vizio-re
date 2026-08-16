# UPnP/DLNA: remote denial of service and parser fuzzing

Unlike the SSID and widget findings, this surface is reachable **remotely from
the LAN** with no user interaction, because the TV actively behaves as a UPnP
control point.

## The TV trusts the network

The master service (`dtv_svc`) broadcasts `M-SEARCH` for MediaServer devices
roughly every 10 seconds and will, in response to any answer on the LAN:

1. fetch the device-description XML from the advertised `LOCATION`,
2. parse it for services (ContentDirectory, ConnectionManager),
3. fetch each service's SCPD document,
4. `SUBSCRIBE` to event URLs, and
5. issue SOAP `Browse` requests and parse the DIDL-Lite XML responses.

Every byte in that chain is attacker-controlled, and the TV performs no
authentication or identity validation of the responding "server."

## Confirmed remote DoS: `Content-Length: -1`

The UPnP event sink on **13000/tcp** mishandles a malformed `Content-Length`.
Testing isolated the exact trigger:

| Content-Length sent | Result |
|---------------------|--------|
| `5` (short) | timeout, service stays alive |
| `999999999` | timeout, service stays alive |
| `0` | **event sink wedged** (persistent) |
| `-1` | **event sink wedged for >60 s after the socket is closed** |

Because the sink stays wedged long after the attacking socket is closed, the
fault is a **state bug inside `dtv_svc`, not a held connection**. The signature
is consistent with an integer-handling error in a read loop: a signed `-1`
length reinterpreted as unsigned becomes ~4 GB, and the server sits trying to
read a body that will never arrive.

**Impact:** a single crafted HTTP request from any LAN device persistently
disables the TV's UPnP event handling; recovery requires a power cycle. This is a
one-packet, unauthenticated, remote DoS.

## DLNA parser fuzzing (methodology and results)

A fake UPnP MediaServer was used to answer the TV's discovery and feed
attacker-controlled DIDL-Lite and device-description XML, rotating a corpus of
malformed cases. Categories exercised:

- **Field overflows** — oversized icon URLs (256 B → 256 KB), titles, resource
  URLs, and `protocolInfo` attributes.
- **Format-string** payloads (`%s%n%x`) in titles and URLs.
- **Integer/count manipulation** — `NumberReturned`/`TotalMatches` set to
  `UINT32_MAX`, `-1`, or mismatched item counts.
- **XML structure** — entity-expansion ("billion laughs"), unclosed tags,
  deeply nested elements, thousands of items, null bytes, CRLF injection.
- **XXE** — external-entity references to local files and to a network URL.

Notable observations:

- **XXE was accepted syntactically (HTTP 200) but not resolved** — no outbound
  fetch and no file content returned, consistent with the XML parser running
  with network/entity resolution disabled.
- **Subscription ID validation works** — bogus SIDs are correctly rejected (412).
- The **`Content-Length: -1` wedge repeatedly cut fuzzing runs short**, since it
  disabled the sink early; large portions of the event-sink corpus (SID/header
  overflows, SEQ edge cases, deep nesting, format strings in properties, large
  bodies) remain **unexplored** because the target had to be power-cycled between
  batches. The negative results here are "not yet reached," not "confirmed safe."

## Why it matters

A 2013 TV that eagerly parses attacker-controlled XML from any LAN peer, with an
outdated `libxml2` and a service that can be wedged by one malformed header, is a
soft target on any shared network. The remotely reachable parser in `dtv_svc` is
the highest-value next target for memory-corruption research on this platform.
