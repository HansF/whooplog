---
title: Flights 007–031 — Twenty-Five Flight Batch
description: A full flash of flights on the aggressive tune, and a battery habit that shows up across all of them.
lead: The batch that prompted the rates change — and the batch during which logging silently stopped.
date: 2026-09-09
weight: 9
toc: true
tags: ["battery", "tune-comparison", "batch"]
craft: ["Crafty"]
firmware: "Betaflight 2026.6.0-alpha"
duration: "~7 min armed across 25 flights"
log_file: "btfl_007.bbl … btfl_031.bbl"
---

## Setup

Twenty-five flights recorded between the
[post-rebuild shakedown](/log/2026-09-08-post-rebuild-shakedown/) and the rates revision.
Extracted as a batch of 31 logs, of which 001–006 belong to the earlier session.

Total armed time across all 31 is about **7 minutes**. Most entries are very short — eight
are under two seconds, which are arm checks rather than flights. Eight logs are long enough
to analyse.

## Which tune each flight actually flew

The blackbox header records the rates in force at the moment logging started, so there is no
need to guess which configuration a flight used:

```bash
head -c 4000 btfl_013.bbl | strings | grep -oP 'rc_expo:\K[0-9,]+'
```

| Logs | `rc_expo` | `rates` | Configuration |
| --- | --- | --- | --- |
| 001–006 | `34,34,27` | `64,64,64` | Original, before any changes |
| 007–031 | `53,40,30` | `55,55,40` | First revision — the aggressive one |
| — | `35,35,25` | `55,55,40` | Current tune — **no flights yet** |

So this entire batch is the evidence behind "too aggressive". Roll expo at 0.53 against pitch
at 0.40 meant the two axes did not respond alike to the same stick movement, which is the
complaint that led to [matching them at 0.35](/reference/rates-actual/).

It also means the current tune has **no flight data at all**. This batch is the before-half of
a clean comparison.

## The flash filled mid-batch

`flash_info` reported `usedSize` equal to `totalSize` — 16,777,216 of 16,777,216 bytes. The
last log written was 031.

{{< callout type="error" >}}
Betaflight's blackbox **does not wrap**. Once full it stops recording, silently, and every
subsequent arm logs nothing. Any flights after 031 produced no data and are unrecoverable.
{{< /callout >}}

This is precisely the failure the
[live log-status element](/reference/osd-layout/#blackbox-log-number-on-screen) was added to
catch — it shows `>` when the flash is full, turning an invisible failure into a visible one.

## Battery — the finding that matters

Five of the eight substantial flights went below 3.00 V, bottoming at 2.83 V.

| Flight | Duration | Min | End | Gap | Reading |
| --- | --- | --- | --- | --- | --- |
| btfl_031 | 35.9 s | 3.17 V | 3.23 V | 0.06 | Flat, but at a safer voltage |
| btfl_017 | 36.4 s | 2.91 V | 2.99 V | 0.08 | **Genuinely flat** |
| btfl_015 | 36.0 s | 3.01 V | 3.12 V | 0.11 | Nearly flat |
| btfl_013 | 39.6 s | 2.96 V | 3.18 V | 0.22 | Mixed |
| btfl_029 | 31.8 s | 2.99 V | 3.34 V | 0.35 | Mostly sag |
| btfl_007 | 59.1 s | 3.15 V | 3.50 V | 0.35 | Healthy |
| btfl_030 | 58.8 s | 2.83 V | 3.21 V | 0.38 | Mostly sag, recovered |
| btfl_028 | 20.2 s | 2.86 V | 3.29 V | 0.43 | Mostly sag, landed with charge left |

The **gap between minimum and end voltage** is what separates two very different situations,
and it is why both are now on the
[post-flight stats screen](/reference/osd-layout/#post-flight-statistics):

- A **large gap** means the pack sagged hard under throttle and recovered once unloaded.
  Unpleasant for the cell but not depletion.
- A **small gap** means the low reading *was* the pack's actual state. 017, 015 and 031 are
  the genuinely damaging ones.

Note that the lowest single reading (2.83 V on btfl_030) is **not** the worst flight — it
recovered to 3.21 V. Flight 017, which never dipped as low, ended 0.08 V above its minimum and
is the one that actually ran the pack down. Judging by minimum voltage alone gets this
backwards.

This is a consistent pattern across a whole session rather than a one-off, which is what
prompted [fixing the warning durations](/reference/osd-layout/#why-the-low-voltage-warning-was-being-ignored) —
the alert was firing on every throttle punch and had become background noise.

## Airframe — healthy

Motor eRPM-per-command spread ranged 5.3 % to 10.0 %.

| Flight | Motor spread | Gyro SD (roll/pitch/yaw) |
| --- | --- | --- |
| btfl_007 | 5.3 % | 21 / 13 / 31 |
| btfl_028 | 5.8 % | 101 / 75 / 42 |
| btfl_015 | 6.3 % | 39 / 24 / 47 |
| btfl_029 | 6.9 % | 12 / 10 / 31 |
| btfl_017 | 7.5 % | 69 / 71 / 66 |
| btfl_030 | 8.1 % | 28 / 50 / 60 |
| btfl_031 | 9.1 % | 13 / 9 / 36 |
| btfl_013 | 10.0 % | 101 / 59 / 57 |

The higher spreads track flying intensity rather than damage — btfl_013 has both the widest
spread and the highest roll gyro SD at 101 °/s. Hard manoeuvring loads the motors unevenly by
definition, so spread is only meaningful compared against flights of similar aggression. A
failing motor would show as a persistent outlier on one motor across *all* flights, gentle
ones included, which is not what appears here.

No motor saturation anywhere. Peak throttle across the batch was 1585 of 2000 — about 66 % of
stick — so there was headroom throughout.

## Open

- **No data yet on the current tune.** The next flights are the after-half of the comparison.
- **`f_yaw` was briefly driven to 663** by a misconfigured adjustment during the OSD profile
  work and restored to its default of 120. Worth confirming yaw feels normal in the air; see
  [the adjrange trap](/reference/aux-modes/).

## Method

Decoded with [`blackbox_decode`](/docs/decode-and-analyze-blackbox/). Voltage figures are the
minimum sample and the mean of the final 0.5 s; motor spread is the range of per-motor
mean `eRPM ÷ mean motor command`, taken over samples above idle throttle only. At
`blackbox_sample_rate = 1/4` the effective rate is ~1 kHz, so nothing here rests on frequency
content above ~500 Hz.
