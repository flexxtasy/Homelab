# Tailscale — Secure Remote Access

Mesh VPN providing secure remote access to the homelab from anywhere, without
port-forwarding or exposing any service to the public internet.

## Why Tailscale (over port-forwarding)

Traditional remote access means forwarding ports on the router, which exposes
services directly to the public internet — a constant attack surface that has to
be patched, firewalled, and monitored.

Tailscale takes a zero-trust approach instead:
- Builds an encrypted mesh network (WireGuard) between my devices.
- **No inbound ports are opened** on the router — nothing is reachable from the
  public internet.
- Devices reach each other over private `100.x.x.x` addresses on the "tailnet".
- Access is tied to authenticated identity, not network location.

For a homelab this is both more secure and simpler to run than port-forwarding.

## Setup

### Proxmox host
```bash
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up            # prints an auth URL — sign in to join the tailnet
tailscale ip -4         # this host's tailnet IP
```

### NixOS workstation (declarative)
```nix
services.tailscale.enable = true;
```
```bash
sudo tailscale up
```

### Laptop (Void Linux)
```bash
sudo xbps-install -S tailscale
sudo systemctl enable --now tailscaled
sudo tailscale up
```

### Phone
Installed the Tailscale app and signed in with the same account.

## Tailnet

All devices authenticate to the same account, forming one private mesh:

| Device | Role |
|--------|------|
| <PROXMOX_HOSTNAME> (Proxmox) | Homelab host |
| NixOS desktop | Workstation |
| Laptop (Void Linux) | Secondary workstation |
| Phone | Mobile access |

## MagicDNS

Enabled MagicDNS (Tailscale admin console -> DNS) so devices are reachable by
name instead of IP — e.g. the Proxmox web UI at `https://<PROXMOX_HOSTNAME>:8006` from any
device on the tailnet, rather than memorizing `100.x.x.x` addresses.

## Result

The Proxmox web UI and homelab services are reachable from any enrolled device,
anywhere (tested from a phone on cellular), with no ports exposed to the public
internet. Remote access is authenticated, encrypted end-to-end, and requires no
changes to router/firewall inbound rules.
