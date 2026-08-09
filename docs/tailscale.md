# Tailscale — Remote Access

Mesh VPN for secure remote access to the homelab from anywhere, without
port-forwarding or exposing services to the public internet.

## Why Tailscale
- No inbound ports opened on the router (nothing exposed to the internet).
- Encrypted end-to-end (WireGuard under the hood).
- Devices reach each other by private `100.x.x.x` addresses on the "tailnet".

## Install (on the Proxmox host)
```bash
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up   # prints an auth URL — open it, sign in to authenticate the host
tailscale ip -4   # shows this host's tailnet IP
```

## Other Devices
- Phone / laptop: install the Tailscale app, sign in with the same account.
- NixOS workstation: `services.tailscale.enable = true;` then `sudo tailscale up`.

## Result
Reach the Proxmox web UI remotely at `https://<tailnet-ip>:8006` from any device
signed into the tailnet — as if on the home LAN.

*(status: in progress)*
