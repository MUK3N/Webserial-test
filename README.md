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

## Fastest free hosting: GitHub Pages

```bash
# from this folder
git init
git add index.html README.md
git commit -m "LevelPro Web Serial test build"
gh repo create levelpro-webserial-test --public --source=. --push
gh api repos/:owner/levelpro-webserial-test/pages -X POST -f "source[branch]=main" -f "source[path]=/"
```
Then open `https://<your-username>.github.io/levelpro-webserial-test/` in
Chrome or Edge, on the machine with the sensor plugged in.

(Netlify Drop, Vercel, or Cloudflare Pages all work the same way if you'd
rather drag-and-drop the folder in a browser instead of using git.)

## Local smoke test first (recommended)

Before deploying, sanity-check the UI locally:

```bash
python3 -m http.server 8080
```
Open `http://localhost:8080` in Chrome/Edge — `localhost` counts as a
secure context, so Web Serial works there too.

## What's real vs. guided-manual in this build

- **Reporting (upload) interval** and **detection interval**: fully
  automatic. Builds the exact downlink-format frame from
  `DF555 LoRaWAN Protocol V3.2` §2.2.1 / §2.2.6
  (`80 02 9999 <cmd> <content> 81`, sent as literal ASCII hex — verified
  against every worked example in the doc) and sends it over the open
  115200 connection. No debug mode needed for this part.
- **Credential read** (DevAddr / DevEui / AppEui): follows the
  `Configuration Manual V2.0` §3 procedure exactly, but as a **guided,
  step-by-step flow** rather than a fully automated one — because two of
  the steps are physical (hold the magnet to the sensor, watch for the
  baud-switch prompt) and the manual only shows screenshots of the
  menu/prompt text, not an exact string to pattern-match against. You
  confirm each physical step in the UI; the app handles the "@" handshake,
  the baud reopen, and parsing the final `AT+ID` reply (`+ID: DevAddr, …`)
  automatically. **AppKey can't be read back** — that's a hardware
  limitation, not a gap in this build.
- A raw serial log + a "send anything" box are always visible, so if a
  prompt or line-ending assumption is slightly off for your actual unit,
  you can see exactly what's happening and intervene by hand.

## Before you rely on it

This hasn't touched a real DF555 — it's built strictly from the two
manuals' documented frames and worked examples, verified byte-for-byte
against every example in the protocol doc, but not yet run against actual
hardware. Start with **reading credentials** (non-destructive) before
sending any **set** command.
