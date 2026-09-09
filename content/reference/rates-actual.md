---
title: Actual Rates
description: Centre sensitivity, max rate and expo per axis, with the CLI variables behind them.
lead: The saved rate profile, and the quantization trap that changes what you type.
weight: 2
toc: true
---

Rate system is `rates_type = ACTUAL`, on rate profile 0.

## Saved values

| Axis | Centre (°/s) | Max (°/s) | Expo | `*_rc_rate` | `*_srate` | `*_expo` |
| --- | --- | --- | --- | --- | --- | --- |
| Roll | 170 | 550 | 0.53 | `roll_rc_rate = 17` | `roll_srate = 55` | `roll_expo = 53` |
| Pitch | 170 | 550 | 0.40 | `pitch_rc_rate = 17` | `pitch_srate = 55` | `pitch_expo = 40` |
| Yaw | 120 | 400 | 0.30 | `yaw_rc_rate = 12` | `yaw_srate = 40` | `yaw_expo = 30` |

Roll and pitch are moderately quick with substantial centre smoothing — more on roll (0.53)
than pitch (0.40). Yaw is deliberately slower and more linear for controlled turns.

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
