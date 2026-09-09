---
title: Process Docs
description: Repeatable procedures for working with the flight controller.
weight: 4
toc: false
---

Procedures that worked, written down with the failure modes that cost time the first time
around. Each one assumes a Linux host and a Betaflight board on USB.

{{< cards >}}
  {{< card link="extract-blackbox-logs/" title="Extracting Blackbox Logs" icon="download"
      subtitle="USB mass-storage mode, mounting, and which files to take." >}}
  {{< card link="decode-and-analyze-blackbox/" title="Decoding a Blackbox Log" icon="chart-bar"
      subtitle="blackbox_decode, and what the decimation warning really means." >}}
  {{< card link="serial-recovery/" title="Recovering a Stuck Serial Link" icon="refresh"
      subtitle="Stuck CLI mode, moving device paths, and ports held by other processes." >}}
  {{< card link="betaflight-mcp-claude-code/" title="Betaflight from Claude Code" icon="terminal"
      subtitle="MCP setup, read whitelisting, and where MSP over USB falls down." >}}
  {{< card link="tx-lua-scripts/" title="Lua Scripts on the Transmitter" icon="device-mobile"
      subtitle="Field tuning from the radio — and which of the two tools uses the wheel." >}}
  {{< card link="elrs-finder/" title="ELRS Lost-Model Finder" icon="search"
      subtitle="RSSI hunting on a mono radio, and the LVGL trap that blanks most finders." >}}
{{< /cards >}}
