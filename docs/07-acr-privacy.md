# ACR beaconing and privacy

Beyond exploitation, the platform is a useful case study in smart-TV telemetry.
The device runs an **Automatic Content Recognition (ACR)** client continuously.

## What it does

A dedicated ACR process (the TVIS / "TV Interactive" client) sends audio/frame
fingerprint data to a remote control server **about every 5.5 seconds**, over
HTTPS using TLS 1.0 / OpenSSL 1.0.0. It fingerprints whatever is on screen,
regardless of input source, and receives configuration back — polling interval,
upload endpoints, and alert/overlay instructions.

## Endpoints and data

- A primary control endpoint (HTTPS, TLS 1.0) receives the fingerprint beacons.
- A debug/frame-upload endpoint is configured in the client.
- Beacon parameters include device/OEM identifiers, chipset info, firmware
  version, region, and language. Config responses can carry alert content,
  including image and action URLs shown as on-screen overlays.

The device identity token used in these beacons is provisioned into the client
and would have to be recovered from the running process's memory. It is **not**
reproduced in this repository.

## Integrity, not confidentiality of intent

The config responses are integrity-checked with a trailing MD5 digest of the form
`MD5(salt + body)`, where the salt is embedded in the client binary. In testing,
corrupting the digest caused the TV to reject the config — so the digest **is**
enforced. That protects the channel's integrity against naive tampering, but it
does nothing for the user's privacy: the TV is designed to report what you watch,
every few seconds, by default.

## Why include this

Two reasons:

1. **Threat-model completeness.** ACR is a always-on outbound data flow with an
   overlay-injection capability (alert image/action URLs). On a controlled
   network its config channel is a surface worth understanding.
2. **Consumer awareness.** This generation of TVs shipped continuous content
   recognition as a default. Documenting it concretely — cadence, transport,
   parameters — is part of the public-interest value of the research.

The transport (TLS 1.0, OpenSSL 1.0.0) is itself obsolete and independently
weak, but the more durable point is architectural: fingerprint-and-report was a
built-in, on-by-default behavior of the platform.
