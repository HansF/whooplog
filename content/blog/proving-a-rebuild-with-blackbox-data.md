---
title: Proving a Rebuild With Blackbox Data
description: How to tell whether a soldering job survived, without guessing.
date: 2026-09-08
tags: ["rebuild", "blackbox", "tooling"]
---

After a rebuild where the iron may have lingered too long, the honest question is whether
anything got cooked. "It flew fine" is weak evidence — a marginal joint or a partially
damaged ESC will fly fine right up until it doesn't.

Blackbox data answers it directly, and the flight needed to collect that answer is 30
seconds of gentle hover.

## The measurement that matters

The useful figure is **eRPM per unit of motor command**, computed per motor over samples
above idle. It is a proxy for how hard each motor has to be driven to produce a given
rotation.

The logic is what makes it decisive. Absolute values drift with battery voltage, prop
condition and air density, so a single motor's ratio tells you little. But all four motors
share those conditions. A damaged power path — cold joint, cracked pad, a phase with extra
resistance, an ESC beginning to desync — makes **one** motor work harder than its
neighbours. It shows up as an outlier, not a uniform shift.

On the [test flight](/log/2026-09-08-post-rebuild-shakedown/) the four ratios came out
1.694, 1.710, 1.757 and 1.779 — a spread under 5%. That's a pass, and it's a pass you can
point at rather than assert.

The companion check is the gyro noise floor. Electrical damage, a compromised ground or a
struggling gyro raise the **broadband** floor rather than adding discrete peaks. All three
axes sat between −38 and −59 dB with no peaks above −15 dB, which rules that out too.

## What the same log surfaced by accident

Two problems, neither related to the rebuild:

The pack had been discharged to **2.48 V**. On 1S that is well past the point where a cell
starts taking permanent damage, and earlier logs showed it trending that way. That is a
flying-habit problem, not a hardware one, and no amount of tuning addresses it.

The current sensor had reported **1320 A** on an earlier flight — an instant step from
1.0 A held for 41 samples, then gone. Physically impossible, and the shape gives it away:
real current ramps, artifacts jump. It's a calibration problem in `ibata_scale` /
`ibata_offset`.

That second one is worth dwelling on, because the general skill transfers. Reading a log
means separating real physics from sensor faults, and the shape of a signal usually tells
you which you're looking at. **Smooth curves are real; instant discontinuities to
implausible values are the instrument lying to you.** Chasing that 1320 A reading as if it
were a short would have wasted an evening.

## The tooling detour

Most of the actual time went to plumbing rather than analysis.

The board runs a 2026.6.0-alpha firmware, and it turns out nearly every MSP binary read
[times out against it](/docs/betaflight-mcp-claude-code/) while CLI text works perfectly.
That inverts the usual approach: the structured binary API is the unreliable one here, and
scraping text is the dependable path.

Working CLI-first then walks straight into a second trap — the board stays in CLI mode until
told otherwise, and a session that ends without `exit` leaves it unresponsive to everything
else until it is physically replugged. Add a browser tab quietly holding the serial port
open, and there are [three distinct failure modes](/docs/serial-recovery/) that all look
like "it stopped responding".

None of that is interesting once you know it, which is precisely why it's written down.
