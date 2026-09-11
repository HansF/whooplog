---
title: Aux Modes and Switches
description: The switch map, and how to read raw aux output correctly.
lead: Resolving a boxId is the step everyone gets wrong.
weight: 6
toc: true
---

## Current switch map

| Slot | Mode | Aux channel | Range | Notes |
| --- | --- | --- | --- | --- |
| 0 | ARM | AUX4 | 1800–2100 | |
| 1 | ANGLE | AUX2 | 900–1300 | Low position of a 3-way switch |
| 2 | HORIZON | AUX2 | 1300–1700 | Mid position — high position is acro |
| 3 | BEEPER | AUX1 | 1700–2100 | |
| 4 | *(cleared)* | — | — | Was OSD DISABLE; removed, see below |
| 5 | FLIP OVER AFTER CRASH | AUX5 | 1700–2100 | Turtle mode |
| 6 | BLACKBOX | AUX1 | 900–2100 | Full span — always active, see below |

Plus one **adjustment** range, which is a separate system from modes:

| Slot | Function | Channel | Range | Notes |
| --- | --- | --- | --- | --- |
| 0 | `ADJUSTMENT_OSD_PROFILE` (CLI value **29**) | AUX3 | 900–2100 | 3-way switch selects [OSD profile](/reference/osd-layout/#switching-profiles-from-a-switch) 1/2/3 |

AUX3 previously carried `OSD DISABLE` on its middle position. That was cleared with
`aux 4 0 0 900 900 0 0` — a zero-width range is how Betaflight represents an empty slot —
because it would have blanked the screen at the same time as selecting profile 2.

AUX2 carries a single 3-position switch: angle at the bottom, horizon in the middle, acro at
the top (acro needs no mode — it is what you get when neither is active).

## Reading raw `aux` output

The CLI prints rows of bare integers:

```text
aux 0 0 3 1800 2100 0 0
aux 5 35 4 1700 2100 0 0
```

The fields are `aux <slot> <boxId> <aux channel> <start> <end> <logic> <linked>`. The aux
channel is zero-indexed, so `0` is AUX1 (RC channel 5).

{{< callout type="error" >}}
**The `boxId` is a `permanentId`, not a position in the `boxId_e` enum.** Resolving it by
counting entries in `src/main/fc/rc_modes.h` gives wrong answers.

The real mapping is the hand-maintained `boxes[]` table in `src/main/msp/msp_box.c`, where
each entry is `{ .boxId = BOXxxx, .boxName = "...", .permanentId = N }`. That table is
**sparse and non-sequential** — it has gaps for removed modes and does not follow enum
declaration order.
{{< /callout >}}

Worked example: `aux 5 35 4 1700 2100` has boxId `35`. Counting through the enum suggests
something entirely unrelated. Grepping the actual table gives the answer:

```bash
grep -oE '\{ *\.boxId *= *BOX[A-Z0-9_]+, *\.boxName *= *"[^"]*", *\.permanentId *= *35 *\}' \
  src/main/msp/msp_box.c
```

```text
{ .boxId = BOXCRASHFLIP, .boxName = "FLIP OVER AFTER CRASH", .permanentId = 35 }
```

Match on `permanentId`, always, against the firmware version actually running — forks and
releases can add modes.

{{< callout type="error" >}}
**`adjrange` has the same trap, one layer deeper.** Its function field is *not* the
`adjustmentFunction_e` enum value, and it is not a box ID either. It indexes
`defaultAdjustmentConfigs[]` in `rc_adjustments.c`, **offset by one**:

```c
#define ADJUSTMENT_FUNCTION_CONFIG_INDEX_OFFSET 1
defaultAdjustmentConfigs[adjustmentRange->adjustmentConfig - ADJUSTMENT_FUNCTION_CONFIG_INDEX_OFFSET]
```

So `ADJUSTMENT_OSD_PROFILE`, enum value 28, is CLI value **29**. Writing the enum value
straight in binds the switch to `ADJUSTMENT_YAW_F` — a *step* adjustment that increments
`f_yaw` continuously while the switch is in range. See
[the OSD page](/reference/osd-layout/#switching-profiles-from-a-switch) for the derivation
script and the incident this caused.
{{< /callout >}}

### Modes versus adjustments

Two different systems, easy to conflate:

- **Modes** (`aux`) turn a boolean on or off — armed, angle, beeper. Identified by
  `permanentId` from `msp_box.c`.
- **Adjustments** (`adjrange`) change a *value* — a PID gain, a profile index. Identified by
  position in `defaultAdjustmentConfigs[]` **plus one**, not by the enum.
  `ADJUSTMENT_MODE_SELECT` functions map switch positions onto discrete values;
  `ADJUSTMENT_MODE_STEP` ones increment continuously, which is why a mistargeted step
  function quietly rewrites a tuning value rather than just doing nothing visible.

If something you want isn't in the mode list, check the adjustment list before concluding it
can't be done from a switch. OSD profile switching exists only as an adjustment.

## Why `small_angle = 180` is correct here

`small_angle` is a pre-arm tilt lock: it refuses arming when the craft is tilted beyond the
given angle. Setting it to 180 disables that check.

That looks alarming in a config diff, and on most builds it would be. Here it is **required**:
turtle mode is mapped to AUX5, and righting an inverted quad means arming while upside down.
A tilt lock would make the feature impossible.

It is unrelated to `angle_limit`, which is the maximum tilt in angle mode. The names are
similar and the settings have nothing to do with each other.

## The BLACKBOX slot is load-bearing

Slot 6 exists so the [live blackbox log number](/reference/osd-layout/#blackbox-log-number-on-screen)
renders on the OSD. Its full-span 900–2100 range is deliberate: assigning `BLACKBOX` to any
range at all makes logging switch-gated, and a full-span range keeps the mode unconditionally
active so behaviour matches an unassigned setup.

Narrowing or removing that line silently changes when logging happens. If blackbox data ever
goes missing, check here first.
