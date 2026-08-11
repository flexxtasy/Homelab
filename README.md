# Homelab

Personal homelab documenting my self-hosted infrastructure, built for hands-on learning in Linux, virtualization, networking, and self-hosting.

## Overview

This repo documents the hardware, services, and configuration of my homelab — what I built, the decisions behind it, and problems I solved along the way.

## Hardware

| Device | Role | Specs |
|---|---|---|
| Lenovo ThinkCentre M720q | Proxmox VE host (hypervisor) | Runs all homelab VMs/containers |
| Seagate 750GB HDD | Bulk storage | ext4, mounted for VM/file storage |
| Custom Desktop | Daily driver / workstation | Ryzen 7 8700F, RTX 5070 Ti, NixOS + Hyprland |
| Laptop | Secondary / backup | Arch Linux |

## Services & Stack

- **Proxmox VE** — bare-metal hypervisor on the ThinkCentre, manages all VMs and containers
- **Tailscale** — mesh VPN for secure remote access (no port-forwarding, no public exposure)
- **Proxmox Firewall** — default-deny host hardening + per-service rules
- **Minecraft Server** — self-hosted vanilla 1.26.2 server (LXC, running)
- **Docker VM** — dedicated Debian VM running containerized services
- **Pi-hole** — network-wide DNS ad/tracker blocking (Docker), delivered to all devices via Tailscale DNS
- **Storage** — Seagate HDD added as Proxmox directory storage

## Network

- Proxmox host: static IP \`<PROXMOX_LAN_IP>\` (LAN)
- Remote admin access via Tailscale (private tailnet, 100.x.x.x addresses)
- Firewall: default-deny on the host; management (web UI, SSH) restricted to the LAN and Tailscale only. The single intentionally-exposed port is Minecraft's 25565, forwarded to a locked-down container that allows only that port.
- DNS: Pi-hole serves as network DNS via Tailscale (works on- and off-LAN, MagicDNS preserved). The ISP router does not allow custom DNS/DHCP, so Tailscale DNS is used instead.

## Documentation

- [Proxmox Setup](docs/proxmox-setup.md)
- [Storage Configuration](docs/storage.md)
- [Tailscale Remote Access](docs/tailscale.md)
- [Firewall & Hardening](docs/firewall.md)
- [Minecraft Server](docs/minecraft-server.md)
- [Docker VM](docs/docker-vm.md)
- [Pi-hole (Docker)](docs/pihole.md)
- [Network-wide Pi-hole via Tailscale DNS](docs/pihole-tailscale-dns.md)
- [Lessons Learned](docs/lessons-learned.md)

## Skills Demonstrated

- Bare-metal hypervisor deployment and management (Proxmox VE)
- Linux system administration (Debian, Arch, NixOS)
- Infrastructure-as-code (NixOS declarative configuration, flakes)
- Containerization: Docker Engine + Compose, running services in a dedicated VM
- Networking: static IPs, VPN mesh networking, DNS filtering, stateful firewall configuration and debugging (pve-firewall, iptables chains)
- Storage: partitioning, filesystems, fstab, persistent mounts
- Troubleshooting and debugging real-world issues (documented in lessons-learned)

This documentation is sanitized — no secrets, private keys, or sensitive network details are committed. See .gitignore.
