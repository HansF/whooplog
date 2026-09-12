---
title: BetaFPV Air65 — 1S Whoop Build Spec
description: The second airframe on this site, and where it differs from the G473 V2 build.
lead: Same board family, different gyro, different tune. Only the pilot settings are shared.
weight: 2
toc: true
craft: ["Air65"]
---

{{< badge content="Betaflight 2026.6.0-alpha" >}}
{{< badge content="MSP API 1.48" >}}

Added 12 September 2026. Configured to match the
[existing craft](/reference/build-betafpvg473-v2-1s/) on everything a pilot feels, and left
at its factory settings on everything specific to its own frame and motors. The port itself
is recorded in the [setup log](/log/2026-09-12-air65-setup/).

## Flight controller

| Item | Value |
| --- | --- |
| Board target | `BETAFPVG473_V2` |
| Manufacturer ID | `BEFH` |
| MCU ID | `002900023235510535303333` |
| MCU | STM32G474, 168 MHz (PLLR-HSI) |
| Gyro / accelerometer | BMI270 (SPI, DMA, locked) |
| OSD | MAX7456, 30 × 13 characters |
| Onboard flash | 16 MB SPI (JEDEC `0x00852018`) — used for blackbox |
| Power | 1S |

{{< callout type="warning" >}}
Both craft report board target `BETAFPVG473_V2`, so the board name cannot tell them apart.
The **MCU ID** is the only reliable discriminator when a backup file turns up without
context. Crafty is `002700443235511330303938`.
{{< /callout >}}

Reported gyro rate in normal operation is ~3.2 kHz with a cycle time around 314 µs and
roughly 36% CPU load. That is lower than the other craft, which runs `pid_process_denom = 2`.

## Different gyro, different firmware

This board carries a **BMI270** where the other carries an ICM42622P. That is not a
variation this site chose — BETAFPV substitute IMUs as supply allows, and ship a matching
firmware build. See [Gyro Substitutions and Firmware](/reference/betafpv-gyro-firmware/)
before flashing anything.

## Motors and tune

| Item | Value |
| --- | --- |
| PID profile name | `GF 1219S` |
| Motor protocol | DShot300, bi-directional |
| Motor poles | 12 |
| `motor_idle` | 550 |
| `dyn_idle_min_rpm` | 30 |
| Board orientation | `align_board_yaw = -135` |

The PID profile, filter stack, board orientation and accelerometer calibration are the
factory values and were deliberately not copied from the other craft. They describe this
frame and these motors.

## Video

| Item | Value |
| --- | --- |
| Band / channel | Raceband 1 (5658 MHz) |
| `vcd_video_system` | `NTSC` |
| `osd_displayport_device` | `MAX7456` |

The two craft are kept on **different Raceband channels** on purpose — Crafty sits on
channel 3 (5732 MHz) — so both can be powered at once without a video clash.

## What is shared

These are pilot settings, not craft settings, and are identical on both airframes:

- [Aux modes and switches](/reference/aux-modes/) — the whole switch map
- [Actual rates](/reference/rates-actual/) — 170/550/0.35 roll and pitch, 120/400/0.25 yaw
- [Throttle curve](/reference/throttle-curve/) — 40/35/44 with an 80% scale limit
- [Crashflip](/reference/crashflip/) and the [OSD layout](/reference/osd-layout/)
