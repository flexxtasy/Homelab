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
      - "80:80/tcp"
    environment:
      TZ: 'America/Chicago'
      FTLCONF_webserver_api_password: '<ADMIN_PASSWORD>'
      FTLCONF_dns_listeningMode: 'all'
    volumes:
      - './etc-pihole:/etc/pihole'
    cap_add:
      - SYS_NICE
    restart: unless-stopped
```

Key points:
- Ports 53 tcp+udp = DNS; port 80 = web admin.
- `FTLCONF_dns_listeningMode: 'all'` lets Pi-hole answer DNS from other
  devices / the Tailscale network (not just localhost).
- `./etc-pihole` volume persists config + blocklists across restarts/updates.
- `restart: unless-stopped` auto-starts on boot/crash.

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

# Dashboard
http://<DOCKER_VM_LAN_IP>/admin
```

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
