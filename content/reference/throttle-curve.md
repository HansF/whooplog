---
title: Throttle Curve and Idle
description: Throttle limit, midpoint, expo and motor idle as saved.
lead: A capped and softened throttle curve for a 1S whoop.
weight: 3
toc: true
---

## Saved values

| Setting | Value | Effect |
| --- | --- | --- |
| `throttle_limit_type` | `SCALE` | Scales the whole output range rather than clipping the top. |
| `throttle_limit_percent` | `90` | Caps usable throttle at 90%, trading unused top-end for headroom and a longer pack. |
| `thr_mid` | `45` | Puts the curve's midpoint slightly below centre stick. |
| `thr_expo` | `35` | Softens response around the midpoint, where hovering happens. |
| `motor_idle` | `400` | 4.00% idle — enough to keep motors turning without creeping. |

Net effect: throttle is capped at 90% and softened around the middle, which makes hover
trim less twitchy on a craft with a lot of thrust relative to its mass.

## SCALE versus CLIP

`SCALE` compresses the entire stick range into the limit, so the curve stays smooth and
full stick still means "maximum available". `CLIP` would instead leave the curve alone and
flatten everything above the limit, which produces a dead zone at the top of the throw.
`SCALE` is the right choice when the goal is a gentler craft rather than a hard ceiling.

## Idle and prop stall

`motor_idle = 400` (4%) is the floor the ESCs are commanded to when armed. Too low and props
can stall on a hard throttle chop, which produces the wobble usually described as propwash.
Too high and the craft creeps on the ground and refuses to descend cleanly.

{{< callout type="info" >}}
This build does not use dynamic idle (`dyn_idle_min_rpm = 0`). Dynamic idle holds a target
**RPM** rather than a fixed throttle percentage and generally handles stall better, but it
requires bidirectional DSHOT and a correct `motor_poles` value. Worth revisiting if
propwash ever becomes the presenting symptom.
{{< /callout >}}
