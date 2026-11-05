# XCAMP Device Configurator — LevelPro test build

A single self-contained `index.html`, no build step, no backend. Skips
login and goes straight to Product Selection, with only LevelPro (DF555)
wired up.

## Why free hosting works fine

Web Serial runs entirely in your browser — the page talks directly to the
USB port via `navigator.serial`, nothing is sent to any server. So where
the HTML file is hosted doesn't matter for the hardware link; the only
requirements are:

1. **A secure context**: HTTPS, or `http://localhost`. Plain `http://` on
   a random domain won't allow Web Serial. `file://` is unreliable across
   Chromium versions — don't rely on it for anything but a first local smoke test.
2. **A Chromium-based browser**: Chrome, Edge, or Opera. Firefox and
   Safari don't implement Web Serial at all.
3. **The browser tab must be open on the computer physically plugged
   into the DF555's USB-TTL adapter.** Hosting elsewhere doesn't change this —
   whoever opens the page needs the device on *their* machine.

So: free static hosting is completely fine.

## Live

Deployed via GitHub Pages: `https://muk3n.github.io/Webserial-test/`
(repo: `MUK3N/Webserial-test`, served from `main` / root). Push to `main`
to update it — Pages rebuilds automatically within a minute or two.

## Local smoke test first (recommended)

Before deploying, sanity-check the UI locally:

```bash
python3 -m http.server 8080
```
Open `http://localhost:8080` in Chrome/Edge — `localhost` counts as a
secure context, so Web Serial works there too.

## Status: verified against real hardware (2026-11-05)

Credential read has been run against an actual DF555 and came back
correct (DevEui/AppEui matched FlexIoT's records, DevAddr matched
ChirpStack). The restart-menu text on the real unit differs from what the
manual's screenshots implied (`Please Select Mode: MB: 1 or 2;XMB: 3 or 4;
HPMB: 5 or 6` rather than a plain numbered list) but sending `3` still
worked as documented, so no code change was needed there.

## What's in this build

- **Reporting (upload) interval**, **detection interval**, **alarm
  thresholds** (full/fire/fall/battery), and **functions** (work mode,
  tilt/fall detection, GPS in uplink, serial echo) — all fully automatic.
  Each builds the exact downlink-format frame from `DF555 LoRaWAN Protocol
  V3.2` §2.2 (`80 02 9999 <cmd> <content> 81`, sent as literal ASCII hex)
  and sends it over the open 115200 connection. No debug mode needed for
  any of these. Every command byte was verified against every worked
  example in the protocol doc before being wired up.
- **Device actions**: Restart sensor (0x09/0x02, same downlink format,
  confirms before sending since it's disruptive).
- **Credential read** (DevAddr / DevEui / AppEui): follows the
  `Configuration Manual V2.0` §3 procedure as a **guided, step-by-step
  flow** — two of the steps are physical (hold the magnet to the sensor,
  watch for the baud-switch prompt), so you confirm each one in the UI.
  The app handles the "@" handshake, the baud reopen, and parsing the
  final `AT+ID` reply (`+ID: DevAddr, …`) automatically.
- **Application key — write-only**: lives under "Device configuration," not
  the credentials card, because it's a *write*, not a read. The DF555's
  LoRaWAN module (confirmed against the LoRa-E5 AT command spec) has no
  command that reads back a stored AppKey — `AT+KEY=APPKEY,"..."` only
  echoes back whatever you just sent, as write confirmation. This is
  deliberate on the manufacturer's side (a stolen device can't leak its
  join key over serial), not a gap in this build. The field:
  - stays disabled until the debug-mode flow (steps 1–4 on the left) has
    switched the connection to 9600 baud,
  - only accepts exactly 32 uppercase hex characters (lowercase is known
    to stall the command on this module),
  - is **cleared the instant you click "Write AppKey"** — the key is never
    kept in the page's state longer than the single send, matching how the
    key itself should be treated (a shared secret you set once and verify
    only by whether the device successfully joins afterward, never read
    back).
  After writing, restart the sensor with the magnet to exit debug mode and
  trigger a fresh OTAA join with the new key.
- A raw serial log + a "send anything" box are always visible, so if a
  prompt, reply format, or line-ending assumption is ever off for your
  actual unit, you can see exactly what's happening and intervene by hand.

## A note on the interval/threshold "set" commands

They write directly to the sensor. If the same device is also managed
through a dashboard (e.g. FlexIoT), changing a value here without updating
the dashboard's record will make the two drift apart — the dashboard just
won't know the device's real setting changed. Worth checking the
dashboard's current value before you press Set, not just the device's.
