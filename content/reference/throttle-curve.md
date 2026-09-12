---
title: Throttle Curve and Idle
description: Throttle limit, midpoint, expo and motor idle as saved.
lead: A capped and softened throttle curve for a 1S whoop.
weight: 3
toc: true
craft: ["Crafty", "Air65"]
---

## Saved values

| Setting | Value | Effect |
| --- | --- | --- |
| `throttle_limit_type` | `SCALE` | Scales the whole output range rather than clipping the top. |
| `throttle_limit_percent` | `80` | Caps usable throttle at 80%, trading unused top-end for headroom and a longer pack. |
| `thr_mid` | `40` | Puts the curve's midpoint below centre stick. |
| `thr_expo` | `35` | Softens response around the midpoint, where hovering happens. |
| `motor_idle` | `400` | 4.00% idle — enough to keep motors turning without creeping. |
| `thr_hover` | `44` | Hover reference point used by altitude-related features. |

Net effect: throttle is capped at 80% and softened around the 40% midpoint, which makes
altitude control precise on a craft with a lot of thrust relative to its mass.

Both the cap and the midpoint were lowered from an earlier setting (90% and 45): the craft
had more top-end than was useful indoors, and moving the midpoint down puts the softened
part of the curve where hover actually sits.

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

## On the Air65

The throttle curve, limit and expo are identical on the
[Air65](/reference/build-air65/). **Motor idle is not:** it runs `motor_idle = 550` against
400 here, which is a factory value for its own motors and was left alone.
