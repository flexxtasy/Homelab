# Proxmox VE Setup

## Host: Lenovo ThinkCentre M720q

Proxmox VE installed bare-metal as the homelab hypervisor.

### Installation

1. Downloaded the **Proxmox VE ISO Installer** from proxmox.com.
2. Flashed to USB (via Ventoy / `dd`) from a Linux workstation — no Windows required.
3. BIOS: enabled virtualization (VT-x/VT-d), set USB boot priority.
4. Installed to the ThinkCentre's internal NVMe (238GB).

### Network Configuration

- **Hostname (FQDN):** `<PROXMOX_HOSTNAME>` on the LAN
- **Static IP:** `<PROXMOX_LAN_IP>/24`
- **Gateway:** `<ROUTER_IP>` (router)
- Static IP chosen so the web UI and services stay at a fixed, known address.
  A DHCP reservation on the router locks it to the host's MAC.

### Accessing Proxmox

- Web UI: `https://<PROXMOX_LAN_IP>:8006` (self-signed cert — expected, accept it)
- Login: `root` + install password
- SSH: `ssh root@<PROXMOX_LAN_IP>`

### Post-Install Notes

- The free version uses the **no-subscription repository** — the enterprise repo
  must be disabled or `apt update` errors. Switch repos before updating:
  ```bash
  apt update
  apt full-upgrade
  ```
- The "no valid subscription" popup on login is normal for the free version.

### VMs vs LXC Containers

Proxmox runs both:
- **LXC containers** — lightweight, low overhead, good for single services (e.g. Minecraft).
- **VMs** — full isolation, needed for non-Linux guests or kernel-level separation.

Homelab services generally use LXC where possible for efficiency.
