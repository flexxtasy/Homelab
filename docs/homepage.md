# Homepage — Service Dashboard

One address to see and reach every homelab service instead of remembering
individual IPs/ports — [gethomepage.dev](https://gethomepage.dev), open
source, YAML-configured.

## Access

- **LAN:** `http://<DOCKER_VM_LAN_IP>:3000`
- **Tailscale:** `http://docker-vm:3000` (or the Docker VM's tailnet IP) from
  any tailnet device — see [Tailscale](tailscale.md)
- Not exposed to the internet — LAN/Tailscale only, consistent with the rest
  of the homelab's [access posture](firewall.md).

## Why Homepage over alternatives

Considered Dashy and Homarr too. Picked Homepage because it has native
widgets that show *live data*, not just bookmarks — a natural fit since
[Pi-hole](pihole.md) and Proxmox already expose APIs, and it auto-integrates
with Docker via the socket (any new container with `homepage.*` labels shows
up automatically).

## Deployment

Runs on the [Docker VM](docker-vm.md) alongside Pi-hole, same pattern
(Compose, `~/<service>/docker-compose.yml`).

`~/homepage/docker-compose.yml`:
```yaml
services:
  homepage:
    container_name: homepage
    image: ghcr.io/gethomepage/homepage:latest
    ports:
      - "3000:3000"
    environment:
      HOMEPAGE_ALLOWED_HOSTS: <DOCKER_VM_LAN_IP>:3000,docker-vm:3000,localhost:3000
    volumes:
      - ./config:/app/config
      - /var/run/docker.sock:/var/run/docker.sock:ro
    restart: unless-stopped
```

Config lives in `~/homepage/config/` on the Docker VM: `settings.yaml`,
`services.yaml`, `widgets.yaml`, `docker.yaml`.

`services.yaml` — groups + entries (Proxmox, Docker VM, Pi-hole, Vaultwarden,
Minecraft):
```yaml
- Infrastructure:
    - Proxmox:
        icon: proxmox.png
        href: https://<PROXMOX_LAN_IP>:8006
        description: Hypervisor
        widget:
          type: proxmox
          url: https://<PROXMOX_LAN_IP>:8006
          username: root@pam!homepage
          password: <PROXMOX_API_TOKEN>
          node: <PROXMOX_HOSTNAME>

    - Docker VM:
        icon: docker.png
        description: Docker host (docker-vm) — no web UI, SSH only
        # no href: port 80 on this host is Pi-hole's admin UI, linking here
        # by mistake sends you to a Pi-hole login instead of "nothing exists"

- Network:
    - Pi-hole:
        icon: pi-hole.png
        href: http://<DOCKER_VM_LAN_IP>/admin
        description: DNS ad-blocking
        widget:
          type: pihole
          url: http://<DOCKER_VM_LAN_IP>
          key: <PIHOLE_ADMIN_PASSWORD>
          version: 6

- Security:
    - Vaultwarden:
        icon: vaultwarden.png
        href: https://docker-vm.<TAILNET_NAME>.ts.net
        description: Password manager (Tailscale-only)

- Games:
    - Minecraft:
        icon: minecraft.png
        href: minecraft://<CONTAINER_LAN_IP>:25565
        description: Vanilla 1.26.2
        widget:
          type: minecraft
          url: http://<CONTAINER_LAN_IP>:25565   # scheme required even though it's not HTTP — see lesson below
```

No widget for the Vaultwarden tile — just a link, since Homepage doesn't have
a native Vaultwarden/Bitwarden widget type. It's reachable from the dashboard
even though it's not on the LAN at all, since the Docker VM (and thus
Homepage itself) is on the same tailnet.

`widgets.yaml` — top info bar (Docker VM's own resource usage + a search box):
```yaml
- resources:
    cpu: true
    memory: true
    disk: /
    label: docker-vm

- search:
    provider: duckduckgo
    target: _blank
```

`docker.yaml` — enables the Docker socket integration (container stats,
label-based auto-discovery for anything deployed later):
```yaml
my-docker:
  socket: /var/run/docker.sock
```

## Status

- [x] Container deployed and healthy (`docker ps` on the Docker VM, port 3000)
- [x] Links to Proxmox, Pi-hole, Minecraft (Docker VM tile is link-less, see note above)
- [x] Minecraft widget live (player count, via server query — no auth needed;
      needed a `http://` scheme prefix, see lesson below)
- [x] Docker widget live (container stats via socket)
- [x] **Proxmox widget** — live. API token created with `--privsep 1` and
      scoped to the read-only `PVEAuditor` role at path `/` (not full root
      privileges) — see [Proxmox Setup](proxmox-setup.md) for the exact commands.
- [x] **Pi-hole widget** — live, using the admin password from Pi-hole's
      compose file (`FTLCONF_webserver_api_password`), plus `version: 6` set
      explicitly (Pi-hole here is v6 — omitting `version` in Homepage risks
      it assuming the old v5 auth flow and erroring).
- [x] **Vaultwarden tile** added under a new "Security" group (link only, no widget)

## Lessons learned

- Homepage ships sensible defaults — only `services.yaml` needs real widget
  credentials to go from "links" to "live dashboard"; the rest works immediately.
- Mounting the Docker socket read-only (`:ro`) is enough for the Docker widget
  and label auto-discovery — no need for full socket access.
- **"Host validation failed" on first load** — Homepage (Next.js under the
  hood) rejects requests whose `Host` header isn't explicitly allowed, as a
  DNS-rebinding protection. Symptom: page loads locally (`curl localhost:3000`
  on the container host) but fails from the LAN IP or Tailscale hostname, with
  `docker logs homepage` showing `Host validation failed for: <host>:3000`.
  Fix: set `HOMEPAGE_ALLOWED_HOSTS` (comma-separated `host:port` list) in the
  compose file's `environment:` to every hostname/IP:port you'll access it
  from, then `docker compose up -d` to recreate the container.
- **A tile's `href` can accidentally collide with another service on the same
  host.** The "Docker VM" tile linked to the Docker VM's bare LAN IP (no
  port = port 80), which is Pi-hole's own admin port on that same VM —
  clicking it looked like a broken/misconfigured Pi-hole rather than an
  unrelated tile. Docker itself has no web UI, so the fix was just removing
  the `href` instead of pointing it somewhere misleading.
- **The Minecraft widget's `url` needs a scheme prefix (`http://`) even though
  Minecraft isn't HTTP.** Docs/examples show `url: <ip>:<port>` with no scheme,
  and that's what caused a silent `TypeError: Invalid URL` server-side (visible
  in `docker logs homepage`, but the browser only ever showed generic "API
  Error: Unexpected error" with no useful detail). Homepage's proxy layer
  calls `new URL()` on every widget's `url` regardless of widget type — Node's
  `URL` constructor requires a scheme, so a bare `host:port` throws before the
  request is even attempted. Adding `http://` (never actually sent as an HTTP
  request — Homepage still does a raw Minecraft server-list-ping) fixed it.
  **Takeaway:** when a Homepage widget fails with a vague "Unexpected error"
  and nothing useful in the browser, check `docker logs homepage` — the real
  exception (e.g. `Invalid URL`) is only logged server-side.
- **Least-privilege API tokens are worth the extra step.** The first Proxmox
  token was created with `--privsep 0` (no privilege separation = inherits
  root's full permissions) purely to get it working fast. Redone properly
  with `--privsep 1` + an explicit `PVEAuditor` (read-only) ACL grant — a
  dashboard widget only needs to *read* cluster status, never write, so it
  should never hold a token that could modify anything.
