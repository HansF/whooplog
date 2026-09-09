---
title: Betaflight Lua Scripts on the Transmitter
description: Installing the TX scripts, and which of the two tools you actually want.
lead: Field tuning without a laptop — once you know that one tool uses the wheel and the other uses the sticks.
weight: 5
toc: true
---

The [betaflight-tx-lua-scripts](https://github.com/betaflight/betaflight-tx-lua-scripts)
project puts PID, rate and filter adjustment on the radio itself. Change a value, fly, change
it again — no laptop, no USB cable, no reconnecting between packs.

They reach the flight controller over the **CRSF telemetry link**, not USB.

## Building

{{< callout type="error" >}}
**Do not copy the repository onto the SD card.** The README is emphatic about this. The card
needs the contents of `obj/`, which `make` generates — the repo root has a different layout
and copying it produces a broken install.
{{< /callout >}}

```bash
make
```

That produces `obj/` containing `SCRIPTS/` and `SOUNDS/` — about 75 files, 476 KB.

The build is lighter than it looks: `bin/build.sh` copies `src/` into `obj/`, generates the
manifest at `obj/SCRIPTS/BF/COMPILE/scripts.lua`, then runs `luac -p` over every file. `-p`
is parse-only, so nothing is precompiled — the scripts ship as `.lua` and compile *on the
radio* at first launch.

{{< callout type="info" >}}
A modern system Lua (5.5) will happily syntax-check these even though EdgeTX runs Lua 5.2.
The check is advisory; parse complaints from a newer Lua are version noise, not real errors.
{{< /callout >}}

## Installing

The install **merges** into directories that already have content — `/SCRIPTS/TOOLS/` will
hold your ELRS and other tools, and `/SOUNDS/en/` will already have a couple of hundred files.

{{< callout type="warning" >}}
Dragging a `SCRIPTS` folder onto the card in a file manager can **replace** the directory
rather than merge into it, taking `elrsV3.lua` and everything else with it. Copy with
explicit merge semantics instead, and back up first.
{{< /callout >}}

{{% steps %}}

### Put the radio in Storage mode

On EdgeTX: **SYS → HARDWARE → USB Mode → Storage (SD)**, then replug. If USB Mode is `Ask`,
choose Storage from the popup on connect.

The radio enumerates as `0483:5720` (Mass Storage) and a FAT partition appears.

### Back up what's there

```bash
cp -r "$SD/SCRIPTS" "$SD/SOUNDS" ~/radio-backup/
```

### Copy with merge semantics

```bash
cp -r obj/SCRIPTS/. "$SD/SCRIPTS/"
cp -r obj/SOUNDS/.  "$SD/SOUNDS/"
sync
```

The trailing `/.` is what makes this merge into the existing directory instead of nesting
inside it or replacing it.

### Verify before unmounting

Confirm the new files arrived **and** the old ones survived:

```bash
test -f "$SD/SCRIPTS/TOOLS/bf.lua"      && echo "installed"
test -f "$SD/SCRIPTS/TOOLS/elrsV3.lua"  && echo "existing tools survived"
ls "$SD/SOUNDS/en" | wc -l              # should be old count + 4
```

### Unmount and reboot the radio

```bash
udisksctl unmount -b /dev/sdb1
```

{{% /steps %}}

## First launch looks like a failure

Opening **TOOLS → Betaflight setup** the first time runs a one-time compile pass and drops
straight back to the TOOLS menu without showing anything. That is expected. Open it a second
time and the real interface appears.

## Two tools, two completely different control schemes

This is the part that wastes time. The install provides two entries, and they are navigated
in entirely different ways.

| | **Betaflight setup** (`bf.lua`) | **Betaflight CMS** (`bfCms.lua`) |
| --- | --- | --- |
| What it is | Native Lua UI with its own pages | A mirror of the FC's on-screen OSD menu |
| Navigate with | **Wheel / buttons** | **The sticks** |
| Use it for | PIDs, rates, filters | Anything in the OSD menu |

**Betaflight setup** handles the events you would expect — `EVT_VIRTUAL_PREV`/`NEXT` to move,
`ENTER` to select, `ENTER_LONG` for menus, `PREV_PAGE`/`NEXT_PAGE` to change page. This is the
one for tuning.

**Betaflight CMS** does not handle rotary input at all. Its entire event loop is:

```lua
if event == radio.refresh.event then      -- ENTER redraws
elseif stickMovement() then                -- ele / ail / rud past ±30
elseif event == EVT_VIRTUAL_EXIT then      -- EXIT closes
```

So turning the wheel in CMS correctly does nothing. Navigate it exactly as you would the OSD
menu in your goggles: **elevator up and down to move through items, aileron and rudder to
change values**, with a decent stick deflection — the threshold is ±30. `ENTER` only refreshes
the screen.

{{< callout type="warning" >}}
A CMS screen that renders but ignores the wheel is **not** broken. It is waiting for stick
input. If you want wheel navigation, you want *Betaflight setup*, not *Betaflight CMS*.
{{< /callout >}}

## What a working CMS proves

Worth noting as a diagnostic in its own right: the CMS pulls its menu content from the flight
controller over MSP-in-CRSF. If the menu draws at all, **MSP request/response is working**.

That is how the [MSP-over-USB timeouts](/docs/betaflight-mcp-claude-code/) were narrowed down
from "this firmware's MSP is broken" to "something in the USB path is broken" — the same board
answering MSP happily over the radio link rules the firmware out.
