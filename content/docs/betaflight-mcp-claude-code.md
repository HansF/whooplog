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

## MSP reads time out over USB

On Betaflight **2026.6.0-alpha** (MSP API 1.48), nearly every MSP binary read times out
through this server, while the CLI text interface works perfectly.

{{< callout type="info" >}}
**Corrected.** This page previously described the timeouts as a firmware-level MSP failure.
That was wrong. The Betaflight TX Lua scripts talk to the same flight controller using MSP
tunnelled over CRSF telemetry, and the [CMS menu renders correctly](/docs/tx-lua-scripts/) —
which it cannot do without working MSP request/response. **The firmware's MSP implementation
is fine.** The fault is confined to the USB path, and most likely to this server's MSP
handling rather than the board.
{{< /callout >}}

| Call | MSP code | Result |
| --- | --- | --- |
| `get_version` (identity) | — | works |
| All CLI text commands (`cli_exec`, `cli_dump`, `cli_diff`, `get`, `set`) | — | work |
| `MSP_STATUS` | 101 | times out |
| `MSP_RAW_IMU` | 102 | times out |
| Battery state | 110 | times out |
| Dataflash summary | 72 | times out |
| `MSP_STATUS_EX` | 150 | times out |

The failure is consistent and survives reconnection, replugging and a fresh boot, so it is not
a transient connection fault. Identity-type calls answering while data calls do not suggests
the server and the firmware disagree about payload format for the affected messages.

{{< callout type="warning" >}}
**Operational rule when using this server: treat CLI text as the reliable transport.**
Anything exposed as a `get_*` MSP tool should be read with `cli_exec "get <name>"` instead.
Board status comes from `status`, not `get_status`; flash usage from `flash_info`, not
`get_dataflash_summary`; log erasure from `flash_erase`, not `erase_blackbox_logs`.
{{< /callout >}}

### Still untested

Raw MSP over USB from a hand-written client has **not** been tried — the one attempt failed
before it opened the port, because the board had re-enumerated on a different `ttyACM` node.
Until that test runs, the boundary is: MSP over CRSF works, MSP over USB *via this server*
does not. Whether a correct USB MSP client would succeed is unknown, and it is the experiment
that would isolate the server from the transport.

## Consequences for CLI-first working

Routing everything through CLI text makes the stuck-CLI failure mode much more likely,
because every read now enters CLI mode. The discipline that follows:

- End every interaction with `exit`.
- Expect the board to report `CLI` in its arming disable flags mid-session — that is normal
  while connected, and clears on reboot.
- A tool that wraps CLI access may itself get confused by a board already in CLI. If
  commands start failing with a stale `#` in the buffer, replug rather than retrying.
