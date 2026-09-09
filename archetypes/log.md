---
title: "Flight NNN — {{ replace .File.ContentBaseName "-" " " | title }}"
description: ""
lead: ""
date: {{ .Date }}
weight: 10
toc: true
tags: []
craft: ""
firmware: ""
duration: ""
log_file: ""
---

## Setup

What this flight was for.

## What changed

Configuration or hardware changes since the previous flight.

## Findings

## Open issues

## Method

Decoded with [`blackbox_decode`](/docs/decode-and-analyze-blackbox/). Raw log is kept
locally and not committed — see that page to reproduce these numbers.
