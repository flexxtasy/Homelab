# Homarr — Service Dashboard

Replaced [Homepage](homepage.md) on 2026-08-18 after a security audit found
Homepage LAN-exposed with no auth, leaking a Proxmox API token and a reused
personal password (see [Lessons Learned](lessons-learned.md) for the full
incident). Homarr does the same job — one page linking every homelab
service — open source, [homarr.dev](https://homarr.dev).

## Access

Tailscale only, same pattern as [Vaultwarden](vaultwarden.md):
`https://docker-vm.<TAILNET_NAME>.ts.net:9443`, proxied via `tailscale serve`
to `127.0.0.1:7575` on the [Docker VM](docker-vm.md). Never bound to the LAN
at all — learned that lesson the first time.

## Deployment

```yaml
services:
  homarr:
    container_name: homarr
    image: ghcr.io/homarr-labs/homarr:latest
    restart: unless-stopped
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./appdata:/appdata
    environment:
      - SECRET_ENCRYPTION_KEY=<GENERATED_VIA_openssl_rand_-hex_32>
    ports:
      - "127.0.0.1:7575:7575"
```

`docker.sock` mounted **read-only** — enough for container auto-detect and
a live Docker widget, no write access needed.

## Integrations

Configured via Homarr's UI (database-backed, unlike Homepage's flat YAML) —
Proxmox (Health widget), Pi-hole (via its own dedicated API app password,
not the admin login password), Docker (auto-detected).

## Lessons learned

- **Proxmox uses a self-signed cert from its own internal CA** — Homarr
  refused the connection until the CA cert (`/etc/pve/pve-root-ca.pem` on
  the Proxmox host) was uploaded under Settings → Certificates and assigned
  to the Proxmox host's IP.
- **API token privilege separation needs its own ACL grant.** Creating the
  token with `--privsep 1` and granting `PVEAuditor` to the *user* isn't
  enough — a privilege-separated token has zero permissions of its own
  until you also run `pveum acl modify / --tokens 'user@realm!tokenid'
  --roles PVEAuditor`. Symptom: the widget "connects" (shows the node name)
  but every stat sits at 0 and VM/LXC counts show 0/0 — confirmed by
  querying the Proxmox API directly with the token and seeing `Permission
  check failed (/nodes/<node>, Sys.Audit)`, rather than guessing from the
  UI alone.
- **Homarr's generic tile "ping" health check is a plain HTTP `fetch()`.**
  It will always show a false-red dot for anything that isn't an HTTP
  server on that port (Minecraft's raw protocol) or isn't reachable from
  Homarr's own container network (Vaultwarden, bound to the Docker VM's
  loopback only). Confirmed via `docker logs homarr`: `fetch failed` for
  Minecraft, `ECONNREFUSED 127.0.0.1:8222` for Vaultwarden. Neither is a
  real problem — just disable the ping/health-check on tiles for services
  that were never meant to answer that kind of check.
- **Homarr's built-in Minecraft widget only queries public servers**, via a
  third-party API (`api.mcsrvstat.us`) — it cannot reach a LAN address
  directly. Only works because this homelab's Minecraft server happens to
  have a live port-forward; point the widget at the public IP, not the LAN
  IP.
- **Containers on the same Docker host can't reach each other by default**
  if they're on separate Compose stacks' isolated networks, even for
  services bound to a real (non-loopback) address — but especially not for
  anything intentionally bound to `127.0.0.1` only (Vaultwarden). Fix:
  explicitly join the dependent container to the target's Docker network in
  its compose file (`networks: - default - <other-stack>_default`), then
  use the other container's *name* as the hostname, on its *internal* port
  — not the host-published port mapping at all. Verified each cross-service
  path directly with `docker exec <container> curl ...` before trusting a
  UI field, since integration UIs don't distinguish "wrong URL" from
  "network can't get there" in their error messages.
