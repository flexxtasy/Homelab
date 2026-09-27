# Local AI Assistant (Hermes)

A self-hosted AI assistant with tool access to this homelab — runs entirely on local
hardware (a local LLM on the workstation), no cloud API, nothing leaving the tailnet.
It replaced an earlier iteration (codenamed *Lyla*, now decommissioned); today it's a
single consolidated bot, **Hermes**.

## What it is
- One local model (a ~27B, quantized to fit a 16 GB GPU) served behind an OpenAI-compatible API.
- Reached three ways, all sharing **one conversation + memory**: a terminal CLI, a
  browser web UI (Tailscale-only), and a Discord bridge (reachable from anywhere).
- A skills + memory system, so it carries context across sessions and can search a
  personal notes vault.

## What it does for the homelab
- **Morning status report + outage alerts.** Deterministic SSH health checks (containers,
  LXCs, NFS, disks, pending updates) — the *facts are gathered by code*, only the wording
  is the model's — posted daily, plus on outages.
- **Approval-gated maintenance.** It can restart a container, apply updates, or reboot a
  host, but every state-changing action needs a **one-time code delivered out-of-band that
  the model never sees**, so it can propose but never self-approve. Read-only status needs
  no code. (See "read freely, act only behind a human-confirmed code" in
  [Lessons Learned](lessons-learned.md).)

## Safety model
- The Discord side has **no shell/file/code tools** — notes are written only through a
  narrow, vault-only note tool. The desktop CLI keeps local tools (single-user).
- On each host the assistant's SSH key is a **forced-command key** that runs only a fixed
  dispatcher; its monitoring account is **read-only** (a root-owned wrapper of fixed
  sub-commands), never general sudo.
- Loads **on demand** — the model spins up when first addressed and unloads when idle, so
  it frees the GPU for other work (e.g. gaming) rather than sitting resident.

No cloud, no keys leaving the box, everything on the tailnet.
