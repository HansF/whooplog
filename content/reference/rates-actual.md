---
title: Actual Rates
description: Centre sensitivity, max rate and expo per axis, with the CLI variables behind them.
lead: The saved rate profile, and the quantization trap that changes what you type.
weight: 2
toc: true
craft: ["Crafty", "Air65"]
---

Rate system is `rates_type = ACTUAL`, on rate profile 0.

## Saved values

| Axis | Centre (°/s) | Max (°/s) | Expo | `*_rc_rate` | `*_srate` | `*_expo` |
| --- | --- | --- | --- | --- | --- | --- |
| Roll | 170 | 550 | 0.35 | `roll_rc_rate = 17` | `roll_srate = 55` | `roll_expo = 35` |
| Pitch | 170 | 550 | 0.35 | `pitch_rc_rate = 17` | `pitch_srate = 55` | `pitch_expo = 35` |
| Yaw | 120 | 400 | 0.25 | `yaw_rc_rate = 12` | `yaw_srate = 40` | `yaw_expo = 25` |

Roll and pitch are matched — same centre sensitivity, same max rate, same expo — giving
smooth, controlled response without one axis feeling softer than the other. Yaw is
deliberately slower and more linear for controlled turns.

### Previous values

The first version of this profile ran noticeably more expo: roll 0.53, pitch 0.40, yaw 0.30.
That proved too aggressive in the air — the heavy centre smoothing on roll made small
corrections feel vague, and the roll/pitch mismatch (0.53 vs 0.40) meant the two axes did not
respond alike to the same stick movement. Reducing expo and matching roll to pitch fixed
both. Rates and max rates were never the problem and are unchanged.

## The quantization trap

{{< callout type="warning" >}}
In ACTUAL mode, centre sensitivity is stored as **`rc_rate × 10 °/s`**, so the representable
values are 10 °/s apart. **165 °/s cannot be expressed** — it rounds to 170 (`rc_rate = 17`).
Max rate is stored the same way as `srate × 10`.
{{< /callout >}}

This is why a value can appear to change on its own after saving. If you enter 165 and the
Configurator later shows 170, nothing went wrong: 165 was never a storable value. The
practical difference is about 3%, which is far below what is perceptible in the air, so the
correct response is to accept the rounding rather than chase it.

The same applies to any ACTUAL-mode figure that is not a multiple of 10 °/s.

## Reading them back

```bash
get roll_rc_rate
get roll_srate
get roll_expo
```

Each returns the current value, the allowed range and the default, and names the rate
profile it belongs to. Changes made with `set` live in RAM only — they are not persisted
until `save`, which also reboots the board.

## On the Air65

Identical on the [Air65](/reference/build-air65/). It shipped on `rates_type = BETAFLIGHT`
with a 110/74/0.30 curve and was converted; the confirmation is that `rates_type` no longer
appears in `diff` at all, because ACTUAL is the firmware default.
