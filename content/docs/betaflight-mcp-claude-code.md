---
title: Driving Betaflight from Claude Code via MCP
description: Wiring an MCP server to the flight controller, and where it breaks on alpha firmware.
lead: Read-only access works well. Most MSP binary reads do not work at all on this firmware.
weight: 4
toc: true
---

## What this is

[`bvandevliet/betaflight-mcp`](https://github.com/bvandevliet/betaflight-mcp) is a
third-party MCP server that exposes a Betaflight flight controller over USB serial — MSP
binary protocol for live data, CLI text for configuration. It auto-generates a
`get_<name>`/`set_<name>` tool pair for each of roughly 375 CLI variables by parsing the
firmware's own `settings.c`.

{{< callout type="info" >}}
Not my project. It is licensed **AGPL-3.0**; the Betaflight firmware it talks to is
GPL-3.0. Credit and licence terms belong to their respective authors.
{{< /callout >}}

## Setup

{{% steps %}}

### Install the plugin

```bash
claude plugin marketplace add bvandevliet/betaflight-mcp
claude plugin install betaflight-mcp@betaflight-mcp
```

This registers both the MCP server (run via `npx`, so it always resolves the latest
published version) and a bundled PID-tuning skill.

### Whitelist the read-only tools

```bash
npx -y -p betaflight-mcp betaflight-mcp-whitelist-reads
```

```text
Found 375 get_* variable tools in variables.js
Added 402 entries to permissions.allow.
Write tools (set_*, cli_save, cli_exec, motor_set, etc.) are NOT whitelisted.
```

{{< callout type="warning" >}}
The project README gives this as `npx -y betaflight-mcp-whitelist-reads`, which **fails with
a 404**. Without `-p`, npx treats the bare argument as a package name and looks for a
published package called `betaflight-mcp-whitelist-reads`, which does not exist. The
`-p betaflight-mcp` flag names the package that *provides* the binary.
{{< /callout >}}

Writes deliberately stay gated — `motor_set` can spin props, so it should require explicit
approval every time.

### Restart and connect

Restart Claude Code so the MCP config loads, then connect by port. The board must not be in
CLI or mass-storage mode — see [Recovering a Stuck Serial Link](/docs/serial-recovery/).

{{% /steps %}}

## The alpha-firmware incompatibility

On Betaflight **2026.6.0-alpha** (MSP API 1.48), nearly every MSP binary read times out,
while the CLI text interface works perfectly.

| Call | MSP code | Result |
| --- | --- | --- |
| `get_version` (identity) | — | works |
| All CLI text commands (`cli_exec`, `cli_dump`, `cli_diff`, `get`, `set`) | — | work |
| `MSP_STATUS` | 101 | times out |
| `MSP_RAW_IMU` | 102 | times out |
| Battery state | 110 | times out |
| Dataflash summary | 72 | times out |
| `MSP_STATUS_EX` | 150 | times out |

The failure is consistent and survives reconnection, replugging and a fresh boot, so it is a
firmware/protocol mismatch rather than a connection fault. Identity-type calls answering
while data calls do not suggests the server and this firmware disagree about payload format
for the affected messages.

{{< callout type="warning" >}}
**Operational rule for this build: treat CLI text as the reliable transport and MSP binary
reads as unavailable.** Anything the MCP server exposes as a `get_*` MSP tool should be read
with `cli_exec "get <name>"` instead. Board status comes from `status`, not `get_status`;
flash usage from `flash_info`, not `get_dataflash_summary`; log erasure from `flash_erase`,
not `erase_blackbox_logs`.
{{< /callout >}}

Worth re-testing once the firmware leaves alpha.

## Consequences for CLI-first working

Routing everything through CLI text makes the stuck-CLI failure mode much more likely,
because every read now enters CLI mode. The discipline that follows:

- End every interaction with `exit`.
- Expect the board to report `CLI` in its arming disable flags mid-session — that is normal
  while connected, and clears on reboot.
- A tool that wraps CLI access may itself get confused by a board already in CLI. If
  commands start failing with a stale `#` in the buffer, replug rather than retrying.
