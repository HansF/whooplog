---
title: CLI Variables for This Build
description: Paste-to-reproduce set lines for the rate profile and throttle curve.
lead: The exact commands to restore this configuration after a wipe.
weight: 4
toc: true
---

These are **this build's** values, not recommendations. They exist so the configuration can
be restored after a firmware flash or compared against a later state.

## Rates and throttle

```bash
# Rate profile 0 — ACTUAL rates
set roll_rc_rate = 17
set roll_srate = 55
set roll_expo = 35
set pitch_rc_rate = 17
set pitch_srate = 55
set pitch_expo = 35
set yaw_rc_rate = 12
set yaw_srate = 40
set yaw_expo = 25

# Throttle curve
set throttle_limit_type = SCALE
set throttle_limit_percent = 80
set thr_mid = 40
set thr_expo = 35
set thr_hover = 44

# Motor idle (4.00%)
set motor_idle = 400

save
```

## Battery alerts

The values that make the low-voltage warning trustworthy rather than constant — see
[OSD Layout](/reference/osd-layout/) for why the durations matter more than the thresholds.

```bash
set vbat_warning_cell_voltage = 350
set vbat_min_cell_voltage = 330
set vbat_max_cell_voltage = 440
set vbat_duration_for_warning = 20
set vbat_duration_for_critical = 20

save
```

{{< callout type="warning" >}}
`save` writes to EEPROM **and reboots the board**. Everything before it lives in RAM only —
disconnecting without saving discards the lot. Conversely, once saved there is no undo, so
capture a `diff all` first if the current state is worth keeping.
{{< /callout >}}

## Capturing current state before changing anything

```bash
diff all
```

`diff` prints only what differs from firmware defaults, which makes it far more readable
than `dump` and is the right thing to paste into a backup file.

{{< details title="Why not `dump`?" >}}
`dump` prints every setting the firmware has, defaults included — thousands of lines. It is
useful when you need to know the literal state of everything, but for backup and restore
`diff` captures exactly the same information about what makes this build different, in a
fraction of the space.
{{< /details >}}

## A note on `set` and CLI mode

Every one of these commands puts the board into CLI mode, and it **stays there** until it
receives `exit` or reboots. A session that ends without `exit` leaves the board unresponsive
to MSP — see [Recovering a Stuck Serial Link](/docs/serial-recovery/).
