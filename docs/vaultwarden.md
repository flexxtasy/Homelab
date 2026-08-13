# Vaultwarden — Self-Hosted Password Manager

Self-hosted, Bitwarden-compatible (open source) password vault. The direct
trigger for adding this was realizing that debugging steps had involved
reading a real admin password in plaintext from a live compose file — goal:
get real secrets out of plaintext compose files and notes and into an
actual vault.

## Access

- **`https://docker-vm.<TAILNET_NAME>.ts.net`** — Tailscale-only, real
  trusted TLS cert (via `tailscale serve`), not reachable from the LAN or the
  internet at all.
- Runs on the [Docker VM](docker-vm.md), container bound to `127.0.0.1:8222`
  only — the only way in is through Tailscale's proxy, there is no direct
  network path to it.

## Why this architecture (no reverse proxy container)

Vaultwarden needs valid HTTPS for browser extensions/mobile apps to trust it
(self-signed certs cause constant friction and normalize clicking through
warnings — bad habit to build). Normally that means running Caddy/nginx to
terminate TLS with a cert you have to renew. Instead this uses **`tailscale
serve`**, which:
- Issues and auto-renews a real cert from Tailscale's CA for the tailnet
  hostname — no cert files to manage.
- Only accepts connections from other tailnet devices — functionally
  equivalent to a firewall rule, but by construction (no exposed port to
  misconfigure).
- One extra process (`tailscaled`, already running for
  [Tailscale](tailscale.md) anyway) instead of a whole extra reverse-proxy
  container.

Net effect: fewer moving parts than a Caddy setup, and arguably more secure
by default.

## Setup

### One-time prerequisites (had to be done manually, not over SSH)
1. `sudo tailscale set --operator=<user>` on the Docker VM — lets that user
   run `tailscale cert`/`serve` without sudo each time.
2. Tailscale admin console (`https://login.tailscale.com/admin/dns`) ->
   **HTTPS Certificates** -> Enable. Off by default per-tailnet; `tailscale
   cert` fails with `"your Tailscale account does not support getting TLS
   certs"` until this is turned on.
3. `sudo apt-get install -y argon2` on the Docker VM — needed to hash the
   admin token.

### Admin token (Argon2id, not plaintext)
Vaultwarden's `/admin` panel is gated by `ADMIN_TOKEN`. Rather than put a raw
password in the compose file, generated a random token and stored only its
Argon2id hash:
```bash
TOKEN=$(head -c 48 /dev/urandom | base64 | tr -d '\n')
SALT=$(head -c 32 /dev/urandom | base64)
HASH=$(printf "%s" "$TOKEN" | argon2 "$SALT" -e -id -k 65540 -t 3 -p 4)
```
The hash (`$argon2id$...`) goes in `ADMIN_TOKEN`; the plaintext `$TOKEN` is
what you actually type into the `/admin` login — not stored anywhere on the
server, so it only exists wherever it was personally saved.

### `~/vaultwarden/docker-compose.yml` on the Docker VM
```yaml
services:
  vaultwarden:
    container_name: vaultwarden
    image: vaultwarden/server:latest
    ports:
      - "127.0.0.1:8222:80"
    environment:
      DOMAIN: 'https://docker-vm.<TAILNET_NAME>.ts.net'
      SIGNUPS_ALLOWED: 'false'  # was 'true' only until the first account existed
      WEBSOCKET_ENABLED: 'true'
      ADMIN_TOKEN: '<ARGON2ID_HASH>'
    volumes:
      - ./vw-data:/data
    restart: unless-stopped
```

**Gotcha:** Docker Compose treats `$` as variable-substitution syntax. An
Argon2 PHC hash is full of `$` characters — each one needs escaping as `$$`
in the YAML or Compose mangles the value silently. See
[Lessons Learned](lessons-learned.md).

### Publish over Tailscale
```bash
tailscale serve --bg 8222
```
Persists across reboots via `tailscaled`'s own state — no systemd unit needed
beyond `tailscaled` itself already running.

## Status

- [x] Container deployed, healthy, bound to localhost only
- [x] Published via `tailscale serve` with a real Tailscale-issued cert
- [x] Admin token generated as an Argon2id hash, plaintext never stored server-side
- [x] Account created
- [x] `SIGNUPS_ALLOWED` flipped to `false` after account creation
- [ ] Existing plaintext secrets (Pi-hole admin password, Proxmox API token)
      migrated in as vault entries
- [ ] **Backups** — the vault's data directory on the Docker VM is the entire
      vault; losing it loses every password. No backup solution exists yet
      anywhere in the homelab (see [Storage](storage.md)) — this raises the
      stakes on that gap. Don't treat this vault as durable until backups exist.

## Lessons learned

See [Lessons Learned](lessons-learned.md) for the consolidated version.
