---
title: A Warning That Is Always On Is Scenery
description: Why an enabled low-voltage alert still let a 1S pack reach 2.48 V.
date: 2026-09-09
tags: ["osd", "battery", "firmware"]
---

The blackbox showed a 1S pack taken down to **2.48 V** — well past the point where a cell
starts taking permanent damage. The obvious explanation is that the low-voltage warning was
switched off.

It wasn't. `use_vbat_alerts = ON`, the battery warning bits were set in `osd_warn_bitmask`,
and the warnings element was visible on screen. Everything was configured correctly and the
pack still got flattened.

## The actual cause

Two settings that weren't in the config diff at all, because they were sitting at defaults:

```text
vbat_duration_for_warning = 0
vbat_duration_for_critical = 0
```

Zero duration means the alert fires the *instant* voltage crosses the threshold. On a 1S
whoop that is close to meaningless, because voltage sags enormously under throttle — the same
log that ends at 2.48 V starts at 3.71 V, and the two numbers are separated by a throttle
punch, not by minutes of flying.

With a 3.50 V warning threshold and zero duration, the warning fires on every punch out from
very early in the pack. By the middle of a flight it is on more than it is off.

At which point it stops being information. You learn, entirely reasonably, that the flashing
thing does not mean anything, and you stop seeing it. The failure wasn't the pilot ignoring a
warning; it was a system that had trained the pilot to ignore it.

The fix is two seconds:

```bash
set vbat_duration_for_warning = 20   # units are 0.1 s
set vbat_duration_for_critical = 20
```

Now the condition has to persist before the alert appears, which filters out sag entirely.
The thresholds never needed changing — 3.50 V and 3.30 V were sane all along. The *timing*
was wrong, and timing is not something you go looking at when the feature is nominally
enabled and appears to be working.

## The general shape of this

Alerting systems fail in two directions, and only one of them is obvious. A warning that
never fires is a bug you find immediately, the first time something goes wrong unannounced. A
warning that fires constantly looks like it is working — it is visibly doing its job — while
being just as useless, and it degrades quietly, because the failure happens in the operator's
attention rather than in the system.

Any threshold alert on a noisy signal needs a duration filter, hysteresis, or both. This one
had `vbat_hysteresis = 1` (0.1 V) and no duration at all, which is enough to stop it
flickering at the boundary but nowhere near enough to survive a 1.2 V sag.

## A second thing, while reading the firmware

Unrelated, but a good trap to know about. Betaflight's CLI prints aux mode assignments as
bare integers:

```text
aux 5 35 4 1700 2100 0 0
```

That `35` is a **`permanentId`**, not an index into the `boxId_e` enum. Resolving it by
counting entries in `rc_modes.h` gives a confidently wrong answer, because the real table in
`msp_box.c` is sparse — it has gaps where modes were removed and does not follow declaration
order. Here, 35 is `FLIP OVER AFTER CRASH`.

That mattered, because it explained a setting that looks like a mistake in isolation:
`small_angle = 180` disables the pre-arm tilt check, which sounds reckless until you notice
turtle mode is on a switch and righting an inverted quad requires arming upside down.

Both of these have the same moral. The config was readable the whole time; what was missing
was knowing which number meant what. Grepping the firmware took a minute and settled both
questions — considerably faster than reasoning about them from memory, and unlike memory, it
was right.
