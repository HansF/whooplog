---
title: whooplog
layout: hextra-home
toc: false
---

{{< hextra/hero-headline >}}
  whooplog
{{< /hextra/hero-headline >}}

{{< hextra/hero-subtitle >}}
  A working reference for one 1S tinywhoop — build spec, rates, blackbox workflow,
  and what the flight logs actually showed.
{{< /hextra/hero-subtitle >}}

<div class="hx:mt-6"></div>

{{< cards cols="2" >}}
  {{< card link="/reference/" title="Reference" icon="table"
      subtitle="Build spec, saved rates, throttle curve and the CLI variables behind them." >}}
  {{< card link="/docs/" title="Process Docs" icon="book-open"
      subtitle="How to pull blackbox logs off the board, decode them, and recover a stuck serial link." >}}
  {{< card link="/log/" title="Flight Log" icon="clipboard-list"
      subtitle="Per-flight findings: motor balance, noise floor, and open issues." >}}
  {{< card link="/blog/" title="Blog" icon="pencil"
      subtitle="Longer write-ups tying a session together." >}}
{{< /cards >}}

<div class="hx:mt-8"></div>

{{< callout type="info" >}}
Everything here describes **one specific airframe** — a BETAFPV G473 V2 1S whoop running a
Betaflight **2026.6.0-alpha** build. Values are recorded so they can be reproduced and
re-checked, not offered as recommendations for other builds.
{{< /callout >}}
