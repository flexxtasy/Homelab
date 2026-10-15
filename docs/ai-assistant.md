# Local AI Assistant (Hermes)

A self-hosted AI assistant with tool access to this homelab — runs entirely on local
hardware (a local LLM on the workstation), no cloud API, nothing leaving the tailnet.
It replaced an earlier iteration (codenamed *Lyla*, now decommissioned); today it's a
single consolidated bot, **Hermes**.

> **Deep dive on the model/inference side:** the GPUs, quantization, context-length and
> VRAM trade-offs, and benchmarks live in a dedicated repo —
> **[Personal-Local-AI-Stack](https://github.com/flexxtasy/Personal-Local-AI-Stack)**.
> This page covers how the assistant plugs into the homelab.

## What it is
- One local model — currently a **~125B-parameter mixture-of-experts** (Qwen3.8-Flash-Next),
  quantized and tuned to run at **256K context with vision on a single 16 GB GPU** (resident
  experts in system RAM + streamed KV cache). Served behind an OpenAI-compatible API.
  Full details: [Personal-Local-AI-Stack](https://github.com/flexxtasy/Personal-Local-AI-Stack).
- Reached two ways that share **one conversation + memory**: a terminal CLI on the workstation,
  and a Discord bridge (reachable from anywhere, but locked to a single allowlisted user).
- A skills + memory system, so it carries context across sessions and can search a personal
  notes vault. Web search is routed through a **self-hosted SearXNG** instance, so queries
  aren't logged by an outside search provider.

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
- **Single-user allowlist.** The Discord bridge answers only its one owner; messages from
  anyone else are ignored. The assistant does have local tool access (terminal/file) for the
  owner's own use — so the protection isn't "no tools," it's **who can reach it** plus the
  controls below.
- **Infrastructure changes can't be self-approved.** Even with tools, any state-changing
  homelab action requires the out-of-band one-time code the model can't see — it proposes,
  a human confirms.
- **Least privilege on every host.** The assistant's SSH key is a **forced-command key** that
  runs only a fixed dispatcher; its monitoring account is **read-only** (a root-owned wrapper
  of fixed sub-commands), never general sudo.
- **Loads on demand** — the model spins up when first addressed / started and unloads to free
  the GPU for other work (e.g. gaming) rather than sitting resident.

No cloud, no keys leaving the box, everything on the tailnet.
