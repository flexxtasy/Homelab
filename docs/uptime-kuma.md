# Uptime Kuma — Status Monitoring

Self-hosted uptime monitor — open source, tracks real historical
up/down status for every homelab service, not just "is it up right now"
like [Homarr](homarr.md)'s dashboard tiles show. Added 2026-08-18.

## Access

Tailscale only, same pattern as everything else in the homelab:
`https://docker-vm.<TAILNET_NAME>.ts.net:8443`, proxied via `tailscale serve`
to `127.0.0.1:3001` on the [Docker VM](docker-vm.md).

## Deployment

```yaml
services:
  uptime-kuma:
    container_name: uptime-kuma
    image: louislam/uptime-kuma:1
    restart: unless-stopped
    volumes:
      - ./data:/app/data
    ports:
      - "127.0.0.1:3001:3001"
```

## Monitors

One monitor per service, all in a "Homelab Services" group: Vaultwarden,
Homarr, Pi-hole, SearXNG, Proxmox, Minecraft (TCP port check, not HTTP).
The AI assistant is intentionally **not** monitored — it lives on a
different host entirely and is Tailscale-only, which would need giving this
container real Tailscale connectivity just for one monitor; not worth it.

Cross-container monitors (Vaultwarden, Homarr) needed the same Docker
network-join trick as [Homarr's own integrations](homarr.md) — see lessons
below.

## Status Page → Homarr integration

Homarr's Uptime Kuma widget doesn't read the main dashboard/API — it reads
a **public Status Page** inside Uptime Kuma, identified by a slug (defaults
to `default`). Created one named "Homelab" (slug `homelab`), added all six
monitors to it, then set Homarr's integration Slug field to `homelab`.

## Lessons learned

- **The connection test hangs instead of erroring when the status page has
  no monitors yet.** Homarr's "Testing connection" just times out silently
  rather than failing fast — looked like a config problem, was actually an
  ordering problem (status page didn't exist yet). Diagnosed by checking
  the actual API response (`/api/status-page/<slug>`) directly rather than
  trusting the UI's spinner.
- **Uptime Kuma has two different "group" concepts that look similar and
  aren't.** A monitor's "Monitor Group" field organizes the *main
  dashboard* sidebar. A status page's groups are a completely separate
  structure you build in the status page editor. Picking the dashboard
  group's name from the status page's "add monitor" search added a nested
  group container instead of the actual monitors — confirmed via the raw
  API response showing `"type":"group"` instead of six individual `"type":
  "http"`/`"port"` entries.
- **The status page editor doesn't autosave.** Losing track of whether an
  edit actually persisted is the single most common failure mode here —
  verified every step against the live `/api/status-page/<slug>` JSON
  rather than trusting what the editor UI displayed, and caught two rounds
  of "looked done, wasn't saved" this way.
- Same container-to-container Docker networking requirement as Homarr:
  Vaultwarden and Homarr are both intentionally bound to the Docker VM's
  loopback only, so Uptime Kuma had to join their respective Compose
  networks (`vaultwarden_default`) to reach them by container name — a
  plain LAN IP doesn't work for services that were deliberately locked down.
