---
title: Decoding a Blackbox Log
description: Turning a .bbl into CSV, and reading the decoder's output honestly.
lead: The decoder reports most of your log as missing. That is usually correct and expected.
weight: 2
toc: true
---

## Installing the decoder

`blackbox_decode` comes from the Betaflight `blackbox-tools` project. On Arch it is in the
AUR:

```bash
paru -S blackbox-tools-git
```

The package is orphaned upstream but still builds cleanly. It provides `blackbox_decode`
and `blackbox_render`.

## Decoding

{{% steps %}}

### Check the log first

```bash
blackbox_decode --stdout btfl_006.bbl > /dev/null
```

This prints the header and statistics without writing files — enough to see the flight
duration and decide whether the log is worth analysing.

### Decode to CSV

```bash
blackbox_decode btfl_006.bbl
```

```text
Decoding log 'btfl_006.bbl' to 'btfl_006.01.csv'...
Log 1 of 1, start 00:11.804, end 00:42.445, duration 00:30.640
```

You get a `.csv` of every logged frame and a small `.event` file marking arm/disarm and
similar events. A 30-second flight produces roughly 8 MB of CSV from a 1.2 MB source.

### Read the statistics block

```text
Looptime            966 avg          171.9 std dev (17.8%)
I frames     961   67.0 bytes avg    64364 bytes total
P frames   29745   37.8 bytes avg  1124433 bytes total
Frames     30706   38.7 bytes avg  1188797 bytes total
Data rate 1002Hz  38997 bytes/s     390000 baud

2 frames failed to decode, rendering 15 loop iterations unreadable.
92175 iterations are missing in total (22983ms, 75.01%).
```

{{% /steps %}}

## "75% of iterations are missing" is not data loss

{{< callout type="info" >}}
That line is a direct consequence of `blackbox_sample_rate = 1/4`, which logs **every
fourth loop iteration** to save flash. Three quarters of iterations were never written by
design, so the decoder correctly reports them missing. Nothing is corrupt.
{{< /callout >}}

What it does mean is a hard ceiling on analysis. With a ~8 kHz gyro decimated by four, the
effective log rate is about **1 kHz**, which by Nyquist puts a **~500 Hz** ceiling on any
frequency claim you can make from the data. That is fine for tracking, oscillation and
gross noise checks, and not enough for detailed motor-noise or filter work.

To trade flight time for resolution, raise the sample rate before the flight:

```bash
set blackbox_sample_rate = 1/1
save
```

The 16 MB flash fills roughly four times faster as a result.

## Deriving numbers from the CSV

The CSV columns include `time (us)`, per-axis `axisP/I/D/F`, `rcCommand`, `setpoint`,
`gyroADC` and `gyroUnfilt`, `accSmooth`, `motor[0..3]`, `eRPM[0..3]`, `vbatLatest`,
`amperageLatest` and flight-mode flags.

Two derived figures used in the [flight log](/log/), both computed over samples where
throttle is above idle:

- **Motor balance** — mean `eRPM[n]` divided by mean `motor[n]`, per motor. Consistent
  ratios across all four mean no cold joints and no desync; a bad joint shows as one
  outlier, not a uniform shift.
- **Gyro noise floor** — an FFT of each `gyroADC` axis, expressed in dB relative to peak.
  Discrete peaks indicate motor or frame resonance; a raised broadband floor indicates an
  electrical or mechanical fault that no amount of tuning will fix.

## Visual analysis

For scrubbing through a log rather than computing over it, the official
[Blackbox Log Viewer](https://github.com/betaflight/blackbox-log-viewer) reads the `.bbl`
directly and shows synchronised gyro, PID, motor and setpoint traces. It is a Vite app —
`npm install && npm start`, then open the local URL.

{{< callout type="warning" >}}
Its `npm install` pulls one dependency from a `git+ssh` GitHub URL, which npm refuses under
default settings with `EALLOWGIT`. Work around it with `npm install --allow-git=all` (the
flag takes `all`/`none`/`root`, not a boolean).
{{< /callout >}}
