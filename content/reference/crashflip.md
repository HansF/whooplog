---
title: Crashflip (Turtle Mode)
description: Righting an inverted quad, and why crashflip_rate is not a speed.
lead: The setting that makes it stop by itself is off by default.
weight: 8
toc: true
---

Crashflip — turtle mode — reverses motor direction so an inverted quad can roll itself
upright. It needs DShot, since only DShot can command a direction change.

{{< callout type="error" >}}
**This spins props on the ground, by design.** Learn the stick directions with props removed
before ever using it in the field. Turtle direction is not intuitive, and discovering it with
props fitted and hands nearby is how people get cut.
{{< /callout >}}

## It is not automatic

The name oversells it. Nothing detects a crash. You land inverted, arm, hold the mode switch,
and push the stick in the direction you want it to roll; the FC reverses the appropriate
motors while you hold it. `crashflip_auto_rearm` only covers re-arming *after* you are upright.

## Settings

| Setting | Default | Here | What it does |
| --- | --- | --- | --- |
| `crashflip_rate` | 0 | **30** | Auto-stop threshold — **not** a speed. See below. |
| `crashflip_auto_rearm` | OFF | **ON** | Re-arms once upright |
| `crashflip_motor_percent` | 0 | 0 | Power given to the opposing motor pair |

Plus the two prerequisites: `motor_pwm_protocol = DSHOT300` and
[`small_angle = 180`](/reference/aux-modes/#why-small_angle--180-is-correct-here), which lets
the craft arm while inverted. The mode itself is on AUX5.

## `crashflip_rate` is an auto-stop

This is the counterintuitive one. From `src/main/flight/mixer.c`:

```c
const float crashflipRateLimit = mixerConfig()->crashflip_rate * 10.0f;
// disable both attenuators if the user's crashflip_rate is zero
if (crashflipRateLimit > 0) {
```

At **0** — the stock default on a non-`USE_RACE_PRO` build — both attenuators are disabled and
motors run for as long as you hold the stick. They stop only when you centre it, leave the
mode, or disarm. That is the familiar turtle-mode failure where the quad flips upright and
then skitters off across the ground still spinning.

Above zero, two attenuators engage:

**Attitude attenuator** (requires an accelerometer):

```c
const float attitudeChangeNeeded = fmaxf(1.0f - fabsf(tiltAngle - tiltAngleAtStart), 0.0f);
crashflipAttitudeAttenuator = attitudeChangeNeeded > halfComplete ? 1.0f
                                                                  : attitudeChangeNeeded / halfComplete;
```

Power falls to zero once the craft has rotated roughly 90° from the attitude it started at. It
flips, then stops itself. The starting attitude is captured when the mode activates, so
arm/disarm resets the reference.

**Rate attenuator:** power tapers as rotation approaches `crashflip_rate × 10` deg/s. At 30
that means attenuation from ~150 deg/s and no power by 300.

{{< callout type="info" >}}
30 is not an arbitrary pick — it is Betaflight's own default under `USE_RACE_PRO`, which also
enables `crashflip_auto_rearm`. Builds without that flag default both to off, which is why
they start disabled here.
{{< /callout >}}

Without an accelerometer only the rate attenuator works, and the firmware comments note the
motors must then be stopped manually.

## `crashflip_motor_percent`

Controls the motors that would push the *wrong* way:

```c
if (motorOutputNormalised < 0) {
    if (mixerConfig()->crashflip_motor_percent > 0) {
        motorOutputNormalised = -motorOutputNormalised * (float)mixerConfig()->crashflip_motor_percent / 100.0f;
    } else {
        motorOutputNormalised = 0;
    }
}
```

At **0**, those motors get nothing and only the pair driving the flip spins. Turtle still works
— this is not a broken configuration, just a gentler one.

Raising it to 20–40 reverses the opposing pair as well, adding leverage. On a whoop the usual
reason to need that is a duct catching on grass. Left at 0 here to keep ground-level motor
power minimal; raise it only if the craft actually fails to roll over.

## Testing

Props **off**, every time, the first time:

1. Place the craft inverted.
2. Hold the AUX5 switch high, then arm.
3. Push roll and pitch in each direction and note which way it tries to rotate.
4. Confirm the motors stop once it has "rotated" ~90° — with props off you are checking that
   output ceases, not that it physically flips.
5. Disarm, lower the switch.

Only then try it with props, outdoors, on a surface you do not mind marking.
