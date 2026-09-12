---
title: Adding a Betaflight OSD Element
description: The integration points, and four things that are not what they look like.
lead: Derived from building a battery voltage sparkline against master, September 2026.
weight: 7
toc: true
---

Adding an OSD element to Betaflight touches eight files. The mechanical part is
documented in a comment block at the top of `src/main/osd/osd_elements.c` and that
comment is accurate, so follow it. What follows is the part the comment does not
cover: four assumptions that look safe and are not.

The worked example is `OSD_BATTERY_VOLTAGE_GRAPH`, a twelve character sparkline of
per-cell battery voltage built on the existing artificial horizon glyphs.

## The AH bar glyphs are upside down relative to intuition

Betaflight ships nine glyphs, `SYM_AH_BAR9_0` through `SYM_AH_BAR9_8` at `0x80`
to `0x88`, each a horizontal line at a different height inside the character cell.
They are the obvious way to draw a graph without shipping a font.

**Index 0 draws the line at the top of the cell, index 8 at the bottom.** So
plotting a value where "high is up" requires inverting the index.

{{< callout type="warning" >}}
Getting this backwards produces a graph that renders, scrolls and looks entirely
plausible while showing voltage upside down. There is no error and no visual tell.
{{< /callout >}}

Two places in the firmware confirm the orientation. In `osdElementArtificialHorizon`
the sub-index `y % 9` increases in the same direction as `elemOffsetY = y / 9`, and
OSD rows increase downward. `osd_nav_map.c` states it outright in a comment and
derives the index from a positive downward offset.

## There is no fast sag-compensated voltage unless the user opted in

The battery getters and their units, all in `src/main/sensors/battery.c`:

| Getter | Returns | Units |
| --- | --- | --- |
| `getBatteryVoltage()` | display-filtered pack voltage | 0.01 V |
| `getBatteryVoltageLatest()` | unfiltered pack voltage | 0.01 V |
| `getLegacyBatteryVoltage()` | rounded pack voltage | 0.1 V |
| `getBatteryAverageCellVoltage()` | filtered, per cell | 0.01 V |
| `getBatterySagCellVoltage()` | sag-filtered, per cell | 0.01 V |

The last one is tempting for anything sag-related and is usually zero.
`voltageSagFiltered` is only computed when `isSagCompensationConfigured()` is true,
which needs a PID profile with `vbat_sag_compensation` above zero and RPM limiting
inactive. Otherwise the filter is never even initialised. Only
`getBatteryVoltageLatest()` is unconditionally populated.

Note also that "latest" means unfiltered, not more frequent. Every one of these
refreshes at the same task rate.

## The OSD task is too slow to sample anything

`TASK_OSD` defaults to 12 Hz, is user-adjustable from 1 to 60 Hz through
`osd_framerate_hz`, and `osdUpdate()` is a state machine that draws **one element
per task call**. Elements can also span several frames by leaving `rendered` false,
so a draw function may be re-entered more than once per frame.

That makes an element's draw function unusable as a sampling hook. Anything
accumulating a time series needs a hook somewhere rate-stable. For voltage the
natural place is `TASK_BATTERY_VOLTAGE`, which runs at 50 Hz and at 200 Hz when sag
compensation is configured. Wrapping it costs one small function in
`src/main/fc/tasks.c`, alongside the existing `taskBatteryAlerts`, and gives exactly
one sample per new ADC reading with no aliasing.

{{< callout type="info" >}}
Sampling in the draw function has a second failure mode beyond rate: it stops
entirely when the element is disabled or scrolled out of the active profile.
{{< /callout >}}

## "Append before OSD_ITEM_COUNT" is not literal

`osd_items_e` ends with several conditional blocks, guarded on `USE_GPS` with flight
plan support, `USE_OSD_NAV_MAP`, `USE_POSITION_HOLD` and `USE_PITOT`. Appending
after them makes the new element's numeric id depend on build flags.

That matters because the Configurator maps wire position N to `DISPLAY_FIELDS[N]`
positionally. A build-dependent id means the graphical OSD editor addresses the
wrong element on some targets. The entry belongs after the last unconditional
member, before the first `#if`.

The frozen constant `OSD_ITEM_COUNT_API_1_46`, currently 80, is a separate concern.
MSP-query OSD consumers such as the DJI air unit get a reply truncated to that
count, so appending is safe for them. `OSD_ITEM_COUNT` must stay at or below 255,
since the MSP reply sends it as a single byte.

## The integration points

Eight files, none optional:

1. `osd/osd.h` — the enum entry, placed as above.
2. `osd/osd.c` — bump the `osdConfig` parameter group version, and `osdElementConfig`
   too, since `item_pos[OSD_ITEM_COUNT]` changes size.
3. `osd/osd_elements.c` — the draw function, an entry in `osdElementDisplayOrder[]`
   and one in `osdElementDrawFunction[]`. Omitting the display order array means the
   element never renders even when visible.
4. `cli/settings.c` — the position setting, before the closing `#endif` of the
   `USE_OSD` block.
5. `cms/cms_menu_osd.c` — a short uppercase entry before the `BACK` row.
6. `mk/source.mk` — any new source file, in **both** OSD lists.
7. `src/test/Makefile` — a new source must be added to every test that links
   `osd_elements.c`, currently `osd_unittest` and `link_quality_unittest`.
8. The Configurator, separately. See below.

`osdAddActiveElements()` only needs touching for sensor-gated elements, and
`osdElementsNeedAccelerometer()` only for ones that read attitude.

Elements render nothing by setting `element->drawElement = false`, not by writing an
empty buffer. The shared `elementBuff` is 32 bytes with no bounds checking anywhere
in the draw path, so assert your width against `OSD_ELEMENT_BUFFER_LENGTH`.

## Testing without an ARM toolchain

The unit tests build with host gcc against a vendored gtest, so element logic is
verifiable with no cross-compiler and no hardware. SITL is not an option here, since
`USE_OSD` is explicitly undefined in its target.

```bash
cd betaflight/src/test
make CC=gcc CXX=g++ test_osd_unittest
```

{{< callout type="warning" >}}
The test Makefile prefers clang and rejects any version outside 7 to 21. A current
Arch clang is 22, so the build fails before compiling anything. Forcing `CC=gcc
CXX=g++` works and is a supported path, though the Makefile warns it is
experimental. Five flight-plan tests fail to compile under gcc regardless of any
local change.
{{< /callout >}}

File-scope statics persist between `TEST_F` cases in one binary, so any module
holding a history buffer needs an explicit reset hook called from `SetUp`.

A useful check on a glyph-based element is to deliberately invert the mapping and
confirm the test fails. An orientation bug is otherwise invisible.

## The Configurator change

Firmware and Configurator are separate repositories, so the graphical OSD editor
does not learn about a new element automatically. Three edits, none of which block
the firmware:

- An entry in `OSD.ALL_DISPLAY_FIELDS` in `src/components/tabs/osd/osd.js`, carrying
  `name`, `text`, `desc`, `defaultPosition`, `draw_order`, `positionable` and a
  `preview` string.
- The same field appended to `OSD.constants.DISPLAY_FIELDS`, **at the position
  matching the firmware enum**. That array is decoded positionally.
- `osdTextElement…` and `osdDescElement…` strings in `locales/en/messages.json`.

## Source

Branch `osd-battery-voltage-graph` on
[HansF/betaflight](https://github.com/HansF/betaflight/tree/osd-battery-voltage-graph),
forked from master at `c68394ac0`.
