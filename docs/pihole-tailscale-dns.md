# Network-wide Pi-hole via Tailscale DNS

How all devices were pointed at Pi-hole for DNS filtering — home **and** away —
while keeping Tailscale MagicDNS (device nicknames) working.

## The problem

The ISP's all-in-one router does not allow changing the DNS server it
hands out to clients, and does not expose a way to disable its DHCP. So:

- Setting the router's DNS to Pi-hole **did not** take effect — devices kept
  using the ISP DNS and bypassed Pi-hole.
- Option "Pi-hole as DHCP server" was blocked because the router's DHCP can't be
  disabled.
- Per-device DNS works but is manual and doesn't cover devices when away from home.

## The solution: point Tailscale at Pi-hole

Because all the important devices are already on a Tailscale tailnet, Tailscale's
DNS can be pointed at Pi-hole. This covers every tailnet device automatically,
works from anywhere, and preserves MagicDNS.

### Steps

1. **Install Tailscale on the Docker VM** (so Pi-hole is reachable over the
   tailnet, not just the LAN):
   ```bash
   curl -fsSL https://tailscale.com/install.sh | sh
   sudo tailscale up      # open the printed URL, log in
   tailscale ip -4        # note Pi-hole's Tailscale IP (100.x.x.x)
   ```

2. **Tailscale admin console** -> https://login.tailscale.com/admin/dns
   - **Nameservers** -> Add nameserver -> Custom -> enter Pi-hole's Tailscale
     IP (`<PIHOLE_TAILSCALE_IP>`)
   - Turn **Override local DNS** ON
   - Keep **MagicDNS** ON

3. Done — all tailnet devices now resolve DNS through Pi-hole.

## Why this is better than the router method

- Works **home and away** (uses Pi-hole's Tailscale IP, reachable anywhere on
  the tailnet — not the LAN-only `192.168.1.x` address).
- Covers all tailnet devices with no per-device config.
- **MagicDNS preserved** — device nicknames still resolve.
- Sidesteps the locked-down ISP router entirely.

## Verify

```bash
# On a client — pi.hole should resolve (proves DNS goes through Pi-hole):
getent hosts pi.hole
#   -> 172.18.0.2   pi.hole

# /etc/resolv.conf shows Tailscale's resolver (which now forwards to Pi-hole):
cat /etc/resolv.conf
#   nameserver 100.100.100.100
#   search <tailnet>.ts.net        <- MagicDNS still active
```

Then check the Pi-hole dashboard Query Log — client devices' traffic appears.

Notes:
- On NixOS, `nslookup` isn't installed by default — use `getent hosts <name>`.
- `resolvectl` errors on NixOS because it uses resolvconf, not systemd-resolved
  — harmless; `getent` is the right tool.

## Reliability note

`FTLCONF_dns_listeningMode: 'all'` in the Pi-hole compose file is required so
Pi-hole accepts DNS queries arriving over the Tailscale interface. Keep the
Docker VM stable, since DNS for the fleet now depends on it (a `1.1.1.1`
fallback can be added at the Tailscale DNS level for resilience).

## Lessons learned

- ISP routers often silently ignore custom DNS and lock DHCP — plan around it.
- Tailscale DNS is an elegant way to get network-wide Pi-hole that also works
  off-LAN, without touching the ISP router.
- Pointing Tailscale's global nameserver at Pi-hole keeps MagicDNS intact.
