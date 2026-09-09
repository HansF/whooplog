---
title: Extracting Blackbox Logs Over USB Mass Storage
description: Getting .bbl files off the onboard SPI flash on Linux.
lead: The board can present its blackbox flash as a USB drive. This is the fastest way to get logs off it.
weight: 1
toc: true
---

The G473 V2 stores blackbox data on 16 MB of onboard SPI flash. Rather than streaming it
over MSP, the board can reboot into USB mass-storage mode and expose that flash as an
ordinary block device.

## Procedure

{{% steps %}}

### Check what's on the flash

In the CLI:

```bash
flash_info
```

```text
Flash sectors=256, sectorSize=65536, pagesPerSector=256, pageSize=256,
totalSize=16777216 JEDEC ID=0x00852018
Partitions:
  0: FLASHFS   0 255
FlashFS size=16777216, usedSize=2306048
```

`usedSize` versus `totalSize` tells you how much is waiting. If they are equal, the flash is
full — see the warning below.

### Reboot into mass-storage mode

```bash
msc
```

The board reboots immediately and disconnects. It will no longer answer CLI or MSP while in
this mode.

### Find the block device

```bash
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT
```

It appears as a small disk with a single partition — typically `/dev/sdb1`, labelled
`BETAFLT`. Confirm the label rather than trusting the letter:

```bash
udisksctl status | grep -i betaflt
```

### Mount it

```bash
udisksctl mount -b /dev/sdb1
```

```text
Mounted /dev/sdb1 at /run/media/hans/BETAFLT
```

{{< callout type="info" >}}
`udisksctl` sometimes reports `Error looking up object for device /dev/sdb1` on the first
attempt, immediately after the device appears. This is a udisks indexing lag, not a failure —
wait a second or two and run it again.
{{< /callout >}}

### Copy the logs you want

{{< filetree/container >}}
  {{< filetree/folder name="BETAFLT" >}}
    {{< filetree/file name="btfl_001.bbl" >}}
    {{< filetree/file name="btfl_002.bbl" >}}
    {{< filetree/file name="btfl_006.bbl" >}}
    {{< filetree/file name="btfl_all.bbl" >}}
    {{< filetree/file name="padding.txt" >}}
  {{< /filetree/folder >}}
{{< /filetree/container >}}

```bash
cp /run/media/hans/BETAFLT/btfl_00*.bbl ~/logs/
```

Take the numbered `btfl_NNN.bbl` files only. `btfl_all.bbl` is just those same logs
concatenated, and `padding.txt` is filler occupying the unwritten remainder of the chip —
copying it wastes a great deal of time for nothing.

### Unmount and return to normal mode

```bash
udisksctl unmount -b /dev/sdb1
```

Then physically unplug and replug the board. Mass-storage mode persists across a soft
reboot; only a real power cycle brings the flight firmware back.

{{% /steps %}}

## The flash does not wrap

{{< callout type="warning" >}}
Betaflight's blackbox **stops logging when the flash is full**. It does not overwrite the
oldest data. Once `usedSize` equals `totalSize`, every subsequent arm records nothing at
all — silently.
{{< /callout >}}

So erasing is a routine part of the cycle, not cleanup:

```bash
flash_erase
```

```text
Erasing, please wait ...

Done.
```

Verify with `flash_info` that `usedSize=0` before flying again. Copy anything you want to
keep **first** — the erase is immediate and unrecoverable.

## Next

[Decode the log](/docs/decode-and-analyze-blackbox/) into something you can analyse.
