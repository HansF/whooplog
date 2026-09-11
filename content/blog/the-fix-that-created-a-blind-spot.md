---
title: The Fix That Created a Blind Spot
description: A low-voltage alarm, two corrections, and a filter that outlived its justification.
date: 2026-09-11
tags: ["battery", "instrumentation", "postmortem"]
---

A 1S pack was being discharged to 2.48 V. The low-voltage alarm was enabled and visible the
entire time.

[The cause](/blog/a-warning-that-is-always-on-is-scenery/) was that
`vbat_duration_for_warning` and `vbat_duration_for_critical` were both zero, so the alert
fired the instant voltage crossed the threshold — including on every throttle punch, on a
craft that sags over a volt under load. It was on more than it was off, so it had stopped
being information.

The fix was two seconds on both. Require the condition to persist, and transient sag stops
triggering it.

That worked. Landing voltages improved across the next session.

## Then the premise expired

Three days later the same craft was diving to **2.43 V** — worse than before — and the alarm
was saying nothing at all.

Nothing had been misconfigured. The alarm was doing precisely what it was told: ignore
excursions shorter than two seconds. What had changed was the hardware. Sag under load had
gone from 0.24 V to 0.39 V across the two sessions, and the confound that would normally
explain that pointed the wrong way — mean peak throttle had *dropped*, from 1505 to 1430. Less
demand, more collapse. That is increased internal resistance, which is what repeated deep
discharge does to a lithium cell.

The earlier over-discharging had damaged the packs. The damaged packs now sag hard enough that
the sag *is* the dangerous event — and the filter installed to ignore sag was, by construction,
ignoring it.

## The shape of the mistake

The filter encoded a claim about the world: *brief excursions below this threshold do not
matter.* That claim was true when it was written. It became false, and nothing in the system's
behaviour marked the transition. A silent alarm looks identical whether the condition is
absent or merely filtered out.

The specific error was collapsing two different questions into one setting. "Is this pack
depleting?" is slow, and transient sag is noise. "Is this dangerous right now?" is fast, and
transient sag is the entire signal. Betaflight exposes those as separate thresholds with
separate durations, and I set both to the same value because I was thinking about one problem.

Now:

```bash
set vbat_duration_for_warning  = 20   # 2.0 s — depleting
set vbat_duration_for_critical = 5    # 0.5 s — dangerous now
```

## What actually generalises

**A filter is a hypothesis, and hypotheses expire.** Every smoothing window, debounce and
minimum-duration encodes an assumption about what is noise. When the underlying system
changes, that assumption can invert without anything appearing to break — you have to go
looking.

**Suppressed and absent look the same from outside.** A threshold alarm that never fires gives
you no way to distinguish "condition never occurred" from "condition occurred and was
filtered". The only reason this surfaced was measuring sag directly rather than trusting the
alarm's silence.

**Check whether your confound moved the other way.** The reflex explanation for more sag is
harder flying. Throttle data said the opposite, which is what turned a plausible story into a
supported one. It is worth asking what *would* have explained the observation innocently, and
then checking whether it actually did.

The instrumentation was right twice and wrong twice, in different ways, about the same
measurement. The logs were the only thing that settled it — which is an argument for recording
more than you think you need, and for occasionally measuring the thing your alarm is supposed
to be watching for.
