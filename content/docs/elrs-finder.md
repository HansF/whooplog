---
title: ELRS Lost-Model Finder
description: An RSSI finder that actually runs on a mono radio, and the fixes it needed.
lead: Hunting a downed quad by signal strength — plus why most published finders won't run on a 128x64 handset.
weight: 6
toc: true
---

## How RSSI hunting works

The quad keeps transmitting telemetry while it has power. Received signal strength rises as
you get closer, so sweeping the radio around gives a warmer/colder reading that walks you in.

{{< callout type="warning" >}}
**Turn TX power down to 10–25 mW before hunting.** At full power everything within tens of
metres reads saturated and the gradient vanishes — precisely when you need it most. Reduce
power further as you close in.
{{< /callout >}}

Two limits worth being honest about:

- **The quad must still be powered and linked.** A disconnected or flat pack means no
  telemetry and no finding. On a 1S whoop that window can be short.
- **It is proximity, not bearing.** Readings are strongly affected by orientation, obstacles
  and ground effect. Within earshot the [DShot beeper](/reference/aux-modes/) is more precise;
  RSSI earns its place beyond that.

## The compatibility trap

Most published ELRS finders target **colour touchscreen** radios and are built on **LVGL**,
which EdgeTX only provides on colour handsets. On a 128×64 mono radio they fail like this:

```lua
if not lvgl then return end      -- drawScreen()
if not lvgl then return 0 end    -- run()
```

Both functions return immediately, so the tool installs cleanly, appears in TOOLS, opens
without error, and shows **a blank screen forever**. No message, no crash. It reads as broken
rather than unsupported.

{{< callout type="info" >}}
Before installing any radio script, check it for `lvgl`, `lcd.RGB()` and coordinates wider
than your screen. A script drawing at `panelX+160` cannot work on a 128 px display. Anything
using only `lcd.drawText`, `lcd.drawRectangle` and `lcd.drawFilledRectangle` is mono-safe.
{{< /callout >}}

## What's installed here

A patched version of `ELRS_Finder.lua` from
[iamsunilchahal/edgetx-lua-scripts-bw](https://github.com/iamsunilchahal/edgetx-lua-scripts-bw)
(**MIT**), which is written for 128×64 B/W and so runs on the
[Radiomaster Pocket](/reference/radio-radiomaster-pocket/).

Signal source, in priority order: `1RSS` (CRSF dBm) → `RSNR` → `RQly`. The reading is
exponentially smoothed, mapped from roughly −110…−40 dBm onto 0–100, and drives Geiger-style
beeps that shorten from 1.2 s to 0.1 s and rise from 600 Hz to 1200 Hz as you approach.

### Changes from upstream

The original is 51 lines and lightly tested. Five fixes:

| Fix | Why |
| --- | --- |
| Handle `EXIT` → `return 1` | Upstream ignores `event` entirely and always returns 0, so the tool never closes |
| `ENTER` toggles beeping | Hunting somewhere you'd rather be quiet |
| Pack voltage + cell detection | Tells you how long the quad will keep transmitting |
| Shortened the footer | Upstream's tip line is ~37 chars ≈ 220 px on a 128 px screen |
| Added a `toolName` marker | TOOLS menu shows a name rather than the filename |

Also seeded the moving average from the first real reading rather than a hardcoded −120,
which otherwise takes ~15 frames to converge on launch, and dropped an unused table.

{{< callout type="info" >}}
The exit omission is an oversight, not house style: the same author's `FieldNotes.lua`
handles `EVT_EXT` properly *and* maps `EVT_ROT_RIGHT`/`LEFT` for rotary radios.
{{< /callout >}}

Event constants are resolved defensively, since the names differ across EdgeTX versions:

```lua
local EVT_EXT = rawget(_G, "EVT_VIRTUAL_EXIT") or rawget(_G, "EVT_EXIT_BREAK") or 0x005B
local EVT_ENT = rawget(_G, "EVT_VIRTUAL_ENTER") or rawget(_G, "EVT_ENTER_BREAK") or 0x0059
```

Cell count divides by **4.5**, not 4.35 — a full 1S LiHV pack sits at 4.35–4.4 V and the
tighter divisor rounds it up to 2S. The per-cell figure blinks below 3.4 V.

## Installing

Source of truth is `betamcp/radio-scripts/SCRIPTS/TOOLS/ELRS_Finder.lua` in the workbench —
edit there, not on the card.

{{% steps %}}

### Syntax check first

```bash
luac -p radio-scripts/SCRIPTS/TOOLS/ELRS_Finder.lua
```

Catches typos before they reach the radio, where the only feedback is a blank screen. A
system Lua 5.5 will check EdgeTX's 5.2 scripts fine for this purpose.

### Copy to the card

Put the radio in Storage mode (**SYS → HARDWARE → USB Mode → Storage**), then:

```bash
cp radio-scripts/SCRIPTS/TOOLS/ELRS_Finder.lua "$SD/SCRIPTS/TOOLS/"
sync
udisksctl unmount -b /dev/sdb1
```

A single file needs no merge care — unlike the
[TX Lua scripts install](/docs/tx-lua-scripts/), which copies whole directories.

{{% /steps %}}

## Using it

**SYS → TOOLS → ELRS Finder.**

| Key | Action |
| --- | --- |
| `EXIT` | Quit |
| `ENTER` | Toggle beeping |

The screen shows the source and raw reading, a strength bar, the smoothed average with a
percentage, and pack voltage with per-cell figure. `No telemetry` blinking means no link —
check the quad still has power.
