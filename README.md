# Homelab

Personal homelab documenting my self-hosted infrastructure, built for hands-on
learning in Linux, virtualization, networking, and self-hosting.

## Overview

This repo documents the hardware, services, and configuration of my homelab —
what I built, the decisions behind it, and problems I solved along the way.

## Hardware

| Device | Role | Specs |
|--------|------|-------|
| Lenovo ThinkCentre M720q | Proxmox VE host (hypervisor) | Runs all homelab VMs/containers |
| Seagate 750GB HDD | Bulk storage | ext4, mounted for VM/file storage |
| Custom Desktop | Daily driver / workstation | Ryzen 7 8700F, RTX 5070 Ti, NixOS + Hyprland |
| Laptop | Secondary / backup | Arch Linux |

## Services & Stack

- **Proxmox VE** — bare-metal hypervisor on the ThinkCentre, manages all VMs and containers
- **Tailscale** — mesh VPN for secure remote access (no port-forwarding, no public exposure)
- **Minecraft Server** — self-hosted vanilla server *(in progress)*
- **Storage** — Seagate HDD added as Proxmox directory storage

## Network

- Proxmox host: static IP `<PROXMOX_LAN_IP>` (LAN)
- Remote access via Tailscale (private tailnet, `100.x.x.x` addresses)
- No inbound ports exposed to the public internet — remote access is Tailscale-only

## Documentation

- [Proxmox Setup](docs/proxmox-setup.md)
- [Storage Configuration](docs/storage.md)
- [Tailscale Remote Access](docs/tailscale.md)
- [Minecraft Server](docs/minecraft-server.md)
- [Lessons Learned](docs/lessons-learned.md)

## Skills Demonstrated

- Bare-metal hypervisor deployment and management (Proxmox VE)
- Linux system administration (Debian, Arch, NixOS)
- Infrastructure-as-code (NixOS declarative configuration, flakes)
- Networking: static IPs, VPN mesh networking, firewall concepts
- Storage: partitioning, filesystems, fstab, persistent mounts
- Troubleshooting and debugging real-world issues (documented in lessons-learned)

---

*This documentation is sanitized — no secrets, private keys, or sensitive network
details are committed. See `.gitignore`.*
