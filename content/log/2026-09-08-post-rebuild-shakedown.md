---
title: Flight 006 — Post-Rebuild Shakedown
description: Verifying a hot-iron rebuild. Airframe passed; two unrelated issues surfaced.
lead: A 30-second hover to check whether the soldering held. It did.
date: 2026-09-08
weight: 10
toc: true
tags: ["hardware-check", "rebuild"]
craft: ["Crafty"]
firmware: "Betaflight 2026.6.0-alpha"
duration: "30.6 s"
log_file: "btfl_006.bbl"
---

## Setup

First real flight after a rebuild during which the board may have been held at soldering
temperature longer than ideal. The question was narrow: did any joint or component suffer,
and is there any electrical damage that would show up under load?

30.6 seconds armed, throttle peaking at 1482 of 2000 (roughly 48% stick). Deliberately
gentle — the goal was a clean, comparable baseline, not performance.

## Findings — airframe passed

### Motor balance

Mean `eRPM` divided by mean motor command, over samples above idle:

| Motor | Mean command | Mean eRPM | Ratio |
| --- | --- | --- | --- |
| 0 | 674 | 1185 | 1.757 |
| 1 | 636 | 1077 | 1.694 |
| 2 | 651 | 1114 | 1.710 |
| 3 | 638 | 1135 | 1.779 |

Spread is under 5%. This is the measurement that answers the rebuild question: a cold joint,
a damaged phase or a partially desyncing ESC shows up as **one motor needing materially more
drive for the same RPM** — an outlier, not a uniform shift. Four consistent ratios means the
power path to every motor is intact.

### Noise floor

FFT of each filtered gyro axis, in dB relative to peak, above 50 Hz:

| Axis | Median noise floor | Discrete peaks |
| --- | --- | --- |
| Roll | −41.9 dB | none above −15 dB |
| Pitch | −38.2 dB | none above −15 dB |
| Yaw | −59.5 dB | none above −15 dB |

No discrete peaks and no raised broadband floor. Broadband elevation is the signature of
electrical interference or a damaged gyro, which was the specific worry after the rework —
it is absent. This log is in fact cleaner than the pre-rebuild flights.

### Saturation

No motor reached its ceiling at any point, and gyro standard deviation stayed modest
(13.9 / 18.7 / 26.2 °/s across roll, pitch, yaw). Nothing suggests a tuning problem.

## Open issues

Neither of these relates to the rebuild.

{{< callout type="error" >}}
**Battery discharged to 2.48 V.** On a 1S pack that is deep over-discharge — normal cutoff
is around 3.0–3.2 V under load. Repeated cycles this low will puff or kill the cell. Earlier
flights showed 2.85–2.92 V, so this is getting worse, not better. Needs a voltage alarm and
shorter flights.
{{< /callout >}}

{{< callout type="error" >}}
**Current sensor reported 1320 A on an earlier flight.** Physically impossible on a 1S
whoop. The signature is diagnostic: an instantaneous step from 1.0 A to 1320.2 A, held for
exactly 41 samples (≈0.43 s), then gone. Real current draw ramps as motors spool; an
instant discontinuity to an implausible constant is an ADC or scaling artifact. Cause is
almost certainly miscalibrated `ibata_scale` / `ibata_offset` — **not** a short or any real
electrical event. Flight 006 itself showed zero such samples.
{{< /callout >}}

## Method

Decoded from `btfl_006.bbl` with [`blackbox_decode`](/docs/decode-and-analyze-blackbox/).

`blackbox_sample_rate` was `1/4`, so the decoder reports ~75% of iterations missing — that
is expected decimation, not corruption. Effective log rate was 1003 Hz, which puts a ~500 Hz
Nyquist ceiling on the frequency analysis. That is why the noise-floor claim above is
scoped to "above 50 Hz" and stops well short of the motor-noise band a full-rate log would
reach.

Raw log kept locally, not committed — see the [log index](/log/) for the rationale.
