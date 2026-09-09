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

| Element | Position | Why |
| --- | --- | --- |
| Blackbox log status | row 1, col 1 | Log number, live — correlates DVR footage to a `.bbl` file |
| Link quality | row 10, col 1 | The only warning you get before a failsafe |
| Warnings | row 10, col 9 | Where the battery alert appears |
| Battery voltage | row 12, col 1 | On 1S this *is* cell voltage |
| Flight timer | row 12, col 23 | The real fuel gauge — see below |
| Flight mode | row 11, col 25 | Required |

Switched off: crosshairs, artificial horizon, AH sidebars, VTX channel, VTX temperature, and
the arming logo (`osd_logo_on_arming = OFF`).

{{< callout type="info" >}}
Elements were hidden by clearing only the visibility bit and **keeping their coordinates**.
Re-enabling any of them is a single `set` that adds 2048 back — nothing was lost. Position
encoding is `pos = (row << 5) | col`, with bit 11 (2048) marking visible in OSD profile 1.
{{< /callout >}}

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
