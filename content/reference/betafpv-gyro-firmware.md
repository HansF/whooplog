---
title: Gyro Substitutions and Firmware
description: Why these boards run a vendor firmware build, and what must not be flashed onto them.
lead: The board you own may not have the gyro the product page lists.
weight: 9
toc: true
craft: ["Crafty", "Air65"]
---

Supplied by BETAFPV with the flight controller, and confirmed against both boards
documented here.

{{< callout type="error" >}}
**Do not flash an official Betaflight release onto these boards.** Betaflight has not yet
released firmware that supports these substituted gyros. The boards ship with a **customized
build of the Betaflight developer firmware**. A stock release can leave the gyro undetected
and the craft unflyable.
{{< /callout >}}

## The substitution

The ICM42688 became scarce, so BETAFPV sourced alternatives and built a compatible firmware
for them. A given board may carry any of:

| Gyro | Seen on |
| --- | --- |
| `ICM42688` | The original part |
| `ICM42622` | [Crafty](/reference/build-betafpvg473-v2-1s/) — reported as ICM42622P |
| `BMI270` | [Air65](/reference/build-air65/) |
| `LSM6DSV16X` | — |
| `LSM6DSK320X` | — |

BETAFPV state that extensive testing confirms the substitutes meet their quality standards.

This is why two boards with the same `BETAFPVG473_V2` target can report different gyros, and
why a filter or PID setting that suits one is not automatically right for the other.

## Identifying what you have

```bash
get status
```

The `GYRO:` and `ACC:` lines name the detected part:

```text
GYRO: (1) BMI270 enabled locked dma
ACC: BMI270
```

## Configurator

Because the firmware is a developer build, the release Configurator may refuse to connect or
show the wrong options. Use the development build at
[master.app.betaflight.com](https://master.app.betaflight.com/).

CLI access over serial is unaffected, which is another reason this site prefers CLI text over
MSP binary reads — see [Driving Betaflight from Claude Code](/docs/betaflight-mcp-claude-code/).

## Blank OSD

If the OSD shows nothing, the display driver and video standard are the first thing to check.
Both default to `AUTO`, and `AUTO` is the state that produces an empty screen here.

```text
set osd_displayport_device = MAX7456
set vcd_video_system = NTSC
save
```

{{< callout type="warning" >}}
These two are **board settings, not pilot settings**. Do not carry them across when copying
an [OSD layout](/reference/osd-layout/) between craft. Copying a layout that had
`vcd_video_system = AUTO` onto the Air65 blanked its screen once already; the
[setup log](/log/2026-09-12-air65-setup/) records it.
{{< /callout >}}
