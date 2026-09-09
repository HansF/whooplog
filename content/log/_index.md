---
title: Flight Log
description: Per-flight findings from decoded blackbox data.
weight: 2
toc: false
---

One entry per flight worth analysing. Each records what the flight was for, what the
decoded log showed, and anything left open.

| Date | Flight | Craft | Duration | Verdict |
| --- | --- | --- | --- | --- |
| 2026-09-08 | [006 — Post-Rebuild Shakedown](2026-09-08-post-rebuild-shakedown/) | G473 V2 1S | 30.6 s | Rebuild sound; two open issues |

{{< callout type="info" >}}
Raw `.bbl` logs are **not committed** to this repository — they are working data, kept
locally. Each entry names its source file, and
[Decoding a Blackbox Log](/docs/decode-and-analyze-blackbox/) documents the exact command
that turns one into the numbers quoted here.
{{< /callout >}}
