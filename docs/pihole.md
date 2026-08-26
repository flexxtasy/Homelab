# Pi-hole (Docker)

Network-wide DNS-based ad/tracker blocking, deployed as the first container on
the Docker VM using Docker Compose.

## Why Compose (not `docker run`)

Docker Compose keeps the configuration declarative and version-controllable —
easy to modify, restart, and reason about, and a better record than a long
`docker run` command.

## docker-compose.yml

Located at `~/pihole/docker-compose.yml` on the Docker VM:

```yaml
services:
  pihole:
    container_name: pihole
    image: pihole/pihole:latest
    ports:
      - "53:53/tcp"
      - "53:53/udp"
      - "127.0.0.1:80:80/tcp"
    environment:
      TZ: 'America/Chicago'
      FTLCONF_dns_listeningMode: 'all'
    volumes:
      - './etc-pihole:/etc/pihole'
    cap_add:
      - SYS_NICE
    restart: unless-stopped
```

Key points:
- Port 53 tcp+udp = DNS, published to the whole LAN — that has to stay
  reachable by every device on the network for Pi-hole to work at all.
- Port 80 (the web admin) is bound to **loopback only**, not the LAN —
  see the lockdown note below for why and what replaced LAN access.
- `FTLCONF_dns_listeningMode: 'all'` lets Pi-hole answer DNS from other
  devices / the Tailscale network (not just localhost).
- `./etc-pihole` volume persists config + blocklists across restarts/updates.
- `restart: unless-stopped` auto-starts on boot/crash.
- **No password in this file.** It used to set the admin password via
  `FTLCONF_webserver_api_password` directly in the compose file — removed
  deliberately, see Lessons Learned.

## Deploy

```bash
mkdir -p ~/pihole && cd ~/pihole
# create docker-compose.yml (above)
docker compose up -d
docker ps          # wait for status "Up (healthy)"
```

## Port 53 conflict (common gotcha)

Pi-hole needs port 53. Debian sometimes runs `systemd-resolved` on 53. Checked with:

```bash
sudo ss -tulpn | grep ':53'
```

Empty output = port free (no conflict on this install). If occupied, free it
before Pi-hole can bind DNS.

## Verify

```bash
# Ask Pi-hole directly to resolve a name
nslookup google.com <DOCKER_VM_LAN_IP>

# Dashboard — Tailscale only, see lockdown note below
https://docker-vm.<TAILNET_NAME>.ts.net:10443/admin
```

## Admin UI locked to Tailscale-only (2026-08-20)

The web admin was originally on `0.0.0.0:80` — reachable by any device on
the LAN, over **plain HTTP**, meaning the login password crossed the
network in cleartext. Same lockdown pattern used everywhere else in this
homelab: `127.0.0.1:80` (loopback only) + `tailscale serve --https=10443`.
DNS (port 53) stays LAN-wide on purpose — that's the one thing here that
*has* to stay open to the whole network.

Also removed the admin password from the compose file's
`FTLCONF_webserver_api_password` env var — it was sitting there in
plaintext, and since it's applied on every container start, it was
silently re-applying an old (and separately compromised) password on every
restart regardless of what was changed via the UI since. Password is now
set once through the UI/CLI (`pihole setpassword`) and persisted in Pi-hole's
own config volume instead.

## Whitelisting (allowing a blocked domain)

When an app/site breaks because a needed domain is blocked:

1. Dashboard -> **Query Log** -> find the domain in red (status "Blocked")
2. Click **Allow** next to it, OR go to **Domains -> Add -> Allow**
3. The app works again; everything else stays blocked

Reactive whitelisting (fix what actually breaks) is cleaner than pre-allowing.

## Lessons learned

- Compose + a persistent volume = settings survive container restarts/updates.
- `dns_listeningMode: all` is required for Pi-hole to serve other devices.
- Some apps (e.g. ISP apps) break when telemetry domains are blocked -> whitelist.
- **A password set via an env var in the compose file gets silently
  re-applied on every restart.** Rotating the password through the UI
  doesn't help if the compose file still has the old value — the next
  restart just puts it back. If a credential ever needs to live in compose
  at all, treat the compose file itself as needing rotation too, not just
  the running instance.
- **Only the *admin UI* needed locking down, not the whole container.**
  Pi-hole serves two genuinely different things — DNS (must stay LAN-wide)
  and a web admin panel (doesn't need to be). Binding the whole container
  to loopback would've broken DNS for the entire network; the fix had to
  be scoped to just the one port, not applied at the service level.
