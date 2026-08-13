# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A documentation-only repo (no code, no build/lint/test tooling): personal
notes on a self-hosted homelab (Proxmox VE hypervisor, Tailscale, Pi-hole,
firewall hardening, Docker VM, Minecraft server). Content is Markdown under
`docs/`, indexed from `README.md`.

## Structure & conventions

- `README.md` — hardware table, services overview, network summary, and a
  linked table of contents into `docs/`. Update it when adding/removing a
  doc or changing the hardware/services list.
- `docs/*.md` — one file per topic/service (e.g. `proxmox-setup.md`,
  `firewall.md`, `pihole.md`). New topics get their own file, linked from
  the README's Documentation list.
- `docs/lessons-learned.md` — a running log of real problems hit and how
  they were debugged, organized by topic with a `**Takeaway:**` line per
  item. When solving a nontrivial issue while writing docs, add an entry
  here rather than only fixing the doc in place.
- Docs favor showing the debugging process (symptoms, root cause, fix,
  takeaway) over just the final config — see `docs/firewall.md` for the
  pattern (design → golden rule → rule tables → a full debugging narrative
  with actual compiled iptables output → takeaways).

## Sanitization rule (important)

This repo is public-facing and **must never contain real secrets or
identifying network details**. Always use placeholders instead of real
values, e.g. `<PROXMOX_LAN_IP>`, `<ROUTER_IP>`, `<CONTAINER_LAN_IP>`.
`.gitignore` also blocks anything matching `*.key`, `*.pem`, `*.env`,
`*password*`, `*secret*`, `*token*`, `authkey*`, `tailscale-auth*`, and a
`private-notes/` directory — real credentials or private notes belong
there, never in `docs/`. When editing or adding docs, check for and
replace any real IPs, hostnames, tokens, or keys with placeholders
consistent with the existing naming style (angle-bracket, SCREAMING_SNAKE
description of what the value is).
