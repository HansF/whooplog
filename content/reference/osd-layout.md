---
title: OSD Layout
description: A minimal OSD built around trustworthy battery information.
lead: Six elements, and the reason the low-voltage warning was previously useless.
weight: 5
toc: true
---

The design brief was narrow: reliable battery information, a minimal screen, and the flight
mode always visible. Everything not serving one of those was switched off.

## What's on screen

Three profiles, selected by the AUX3 3-position switch:

| Element | Position | P1 Full | P2 Minimal | P3 Clean | Why |
| --- | --- | :-: | :-: | :-: | --- |
| Warnings | row 10, col 9 | ● | ● | ● | Where the battery alert appears |
| Battery voltage | row 12, col 1 | ● | ● | | On 1S this *is* cell voltage |
| Flight mode | row 11, col 25 | ● | ● | | Required |
| Blackbox log status | row 1, col 1 | ● | | | Log number, live — correlates DVR footage to a `.bbl` |
| Link quality | row 10, col 1 | ● | | | The only warning you get before a failsafe |
| Flight timer | row 12, col 23 | ● | | | The real fuel gauge — see below |

Switched off in every profile: crosshairs, artificial horizon, AH sidebars, VTX channel, VTX
temperature, current draw, and the arming logo (`osd_logo_on_arming = OFF`).

{{< callout type="warning" >}}
**Warnings stay visible even in the "clean" profile.** A genuinely blank screen means no
low-voltage alert, and this craft has a [history of deep
discharge](/log/2026-09-08-post-rebuild-shakedown/). Clean video is not worth losing the one
indicator that protects the pack.
{{< /callout >}}

## Position encoding

```text
pos = (row << 5) | col        bits 0-10
bit 11 (2048)  visible in profile 1
bit 12 (4096)  visible in profile 2
bit 13 (8192)  visible in profile 3
```

An element appears in every profile whose bit is set, so the values above are a base position
plus the sum of its profile bits:

| Element | Base | Profile bits | Value |
| --- | --- | --- | --- |
| `osd_warnings_pos` | 329 | 2048+4096+8192 | **14665** |
| `osd_vbat_pos` | 385 | 2048+4096 | **6529** |
| `osd_flymode_pos` | 377 | 2048+4096 | **6521** |
| `osd_log_status_pos` | 33 | 2048 | **2081** |
| `osd_link_quality_pos` | 321 | 2048 | **2369** |
| `osd_tim_2_pos` | 407 | 2048 | **2455** |
| `osd_current_pos` | 384 | none | **384** (hidden) |

{{< callout type="info" >}}
Hiding an element means clearing its profile bits and **keeping the coordinates**, so
restoring it is a single `set` that adds the bits back. Nothing is lost. Note that an element
with only bit 13 set is visible in profile 3 *only* — `osd_current_pos` was `8576` before this
change, which would have put the [broken current sensor's](/log/2026-09-08-post-rebuild-shakedown/)
readings on the clean-video profile specifically.
{{< /callout >}}

## Switching profiles from a switch

There is **no RC mode box for OSD profiles** — `msp_box.c` has only `OSD DISABLE`
(permanentId 19). Profile selection is an *adjustment*, not a mode:

```bash
adjrange 0 0 2 900 2100 29 2 0 0
```

Fields are `<index> <unused> <range channel> <start> <end> <function> <select channel> <center> <scale>`.
Range channel and select channel are both AUX3 (index 2); the 900–2100 range means the
adjustment is always enabled, and AUX3's own position selects the value.

`ADJUSTMENT_OSD_PROFILE` is `ADJUSTMENT_MODE_SELECT` with `switchPositions = 3`, and
`rc_adjustments.c` divides the channel evenly:

```c
const uint16_t rangeWidth = (2100 - 900) / switchPositions;          // 400
const uint8_t position = (constrain(rcData[ch], 900, 2099) - 900) / rangeWidth;
```

| AUX3 | PWM | position | Profile |
| --- | --- | --- | --- |
| Low | 900–1299 | 0 | 1 |
| Mid | 1300–1699 | 1 | 2 |
| High | 1700–2099 | 2 | 3 |

which lines up with a standard 3-way switch's ~1000/1500/2000 output.

{{< callout type="error" >}}
**The CLI function number is the enum value plus one, and getting it wrong is destructive.**

`ADJUSTMENT_OSD_PROFILE` is `28` in `adjustmentFunction_e` — but the CLI value is **29**,
because the stored number indexes a *different* array with an offset:

```c
#define ADJUSTMENT_FUNCTION_CONFIG_INDEX_OFFSET 1
defaultAdjustmentConfigs[adjustmentRange->adjustmentConfig - ADJUSTMENT_FUNCTION_CONFIG_INDEX_OFFSET]
```

Reading `28` off the enum and writing it into `adjrange` binds the switch to
`ADJUSTMENT_YAW_F` instead. That is a **step** adjustment: it does not select a value, it
*increments* one, every pass, for as long as the switch sits in range. Doing this drove
`f_yaw` from its default of 120 to 663 before it was noticed — silently, with no error, and
it was only caught because two reads seconds apart disagreed.

Derive it from the array that the firmware actually indexes, not the enum:

```bash
python3 - <<'PY'
import re
src = open('src/main/fc/rc_adjustments.c').read()
body = re.search(r'defaultAdjustmentConfigs\[.*?\]\s*=\s*\{(.*?)\n\};', src, re.S).group(1)
for i, f in enumerate(re.findall(r'\.adjustmentFunction\s*=\s*(ADJUSTMENT_[A-Z0-9_]+)', body)):
    print(f"CLI value {i+1:>3}  {f}")
PY
```

Then sanity-check the result against something you can observe. Step-mode functions are the
dangerous ones; select-mode functions merely set a value and are harmless if mistargeted.
{{< /callout >}}

Profile support must be compiled in — check with `get osd_profile`, which should report a
range of 1–3. A build without `USE_OSD_PROFILES` has only one profile.

`OSD DISABLE` was removed from AUX3 (`aux 4 0 0 900 900 0 0`) because it sat on the middle
position and would have blanked the screen while also selecting profile 2. Profile 3 serves
that purpose better, since it keeps warnings.

## Why the low-voltage warning was being ignored

This is the finding that mattered most.

`vbat_duration_for_warning` and `vbat_duration_for_critical` were both **`0`**, meaning the
alert fires the instant voltage crosses the threshold — including on every throttle punch. A
1S whoop sags enormously under load: the [flight 006 log](/log/2026-09-08-post-rebuild-shakedown/)
shows 3.71 V dropping to 2.48 V. With a 3.50 V threshold and zero duration, the warning was
almost certainly flashing from early in every pack.

A warning that is always on is not a warning. It is scenery.

```bash
set vbat_duration_for_warning = 20   # 2.0 s, units are 0.1 s
set vbat_duration_for_critical = 20
```

Now the alert only appears when the pack is genuinely depleted, not when you are on the
throttle. The thresholds themselves (3.50 V warn, 3.30 V critical) were always sane — the
timing was the broken part.

{{< callout type="warning" >}}
`vbat_sag_compensation` is deliberately left at **0**. It keeps throttle authority constant
as the pack drains, which also removes the natural throttle-softening that tells you the
battery is going. On a craft with a history of over-discharge, that cue is worth keeping.
{{< /callout >}}

## Post-flight statistics

`osd_stat_bitmask = 8676` selects: flight timer, **min battery**, **end battery**, battery,
min RSSI, and blackbox log number.

The important pair is **min battery** and **end battery**:

- **Min** is the lowest loaded voltage, sag included.
- **End** is the resting voltage after landing.

The *gap between them* is the diagnostic. A large gap means the pack sagged hard under load
but recovered — normal. A small gap means you genuinely ran it flat.

Max current was removed from the stats: this board's current sensor is miscalibrated and
produces impossible readings (see the flight log), so any number derived from it is noise.

## Warnings

`osd_warn_bitmask = 140287` — arming disable, all four battery states, visual beeper,
crashflip, ESC failure, core temperature, failsafe, launch control, link quality and load.

The GPS-rescue and position-hold warning bits were cleared. This craft has no GPS, so those
conditions can never be meaningful; leaving them enabled is pure screen noise.

## Video system

`vcd_video_system = AUTO`.

{{< callout type="warning" >}}
The character grid is determined by the **camera's** signal, not the goggles. NTSC gives
30×13, PAL gives 30×16. Forcing PAL with an NTSC camera misaligns the OSD. `AUTO` follows
whatever the camera outputs and cannot break, so it is the right setting unless you have a
specific reason to pin one.
{{< /callout >}}

## Blackbox log number on screen

`OSD_LOG_STATUS` is a live element — unlike `BB LOG NUM`, which only appears on the
post-flight stats screen. It renders the blackbox symbol plus the current log number from
arming onward, so DVR footage carries the name of its own `.bbl` file. It also shows:

- `!` — the blackbox device is not working
- `>` — **the flash is full**

That second one matters because [the flash does not wrap](/docs/extract-blackbox-logs/): it
silently stops logging when full. This turns an invisible failure into a visible one.

### The trade-off

The element only draws when the `BLACKBOX` RC mode is active, which means assigning it to a
switch. From `blackbox.c`:

```c
if (blackboxModeActivationConditionPresent && !IS_RC_MODE_ACTIVE(BOXBLACKBOX) && ...) {
    blackboxSetState(BLACKBOX_STATE_PAUSED);
}
```

`blackboxModeActivationConditionPresent` becomes true as soon as *any* aux range is assigned
to `BLACKBOX`. From that moment **logging is gated by that switch** — if the mode goes
inactive, logging pauses. With nothing assigned, logging simply always runs when armed.

The mitigation is a range covering the full span, so the mode is unconditionally active:

```bash
aux 6 26 0 900 2100 0 0
set osd_log_status_pos = 2081
```

Use an aux channel that **actually exists**. An unused high channel may read outside
900–2100, which would silently pause logging — the exact failure being avoided.

{{< callout type="error" >}}
Because this condition now exists, restoring an older config dump that lacks the `aux 6` line
would leave `BLACKBOX` assigned-but-narrow or unassigned in a way that changes logging
behaviour. If logging ever stops unexpectedly, check the aux map first.
{{< /callout >}}
