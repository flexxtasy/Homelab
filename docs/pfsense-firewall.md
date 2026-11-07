# pfSense — One-Armed Lab Firewall VM

A pfSense Community Edition VM added to the Proxmox host as a **lab firewall**
with its own private network segment, subnet-routed over Tailscale so the lab
is reachable from away. Built Nov 2026; the host-side firewall story lives in
[firewall.md](firewall.md) — this is the guest-side appliance.

## Why "one-armed" (the constraint that decided the design)

The WAN is the ISP's mobile gateway behind **CGNAT**: no bridge mode, no port
forwarding, no control of NAT. A VM firewall therefore cannot replace the edge
router — inbound from the internet is impossible no matter what runs here. The
design that *is* possible:

- **WAN NIC** on the existing uplink bridge (`vmbr0`), DHCP lease from the
  gateway (`<PFSENSE_WAN_IP>`).
- **LAN NIC** on a new Linux bridge `vmbr1` with **no bridge ports attached**
  — a VMs-only private segment `<LAB_SEGMENT>` (lab-internal addressing, never
  routed outside the home).

Double-NAT ("one-armed"), but nothing existing changes: no gateway or DHCP
edits, Pi-hole, Tailscale and NFS all untouched, and the management uplink
`vmbr0` is untouched, so management access cannot be lost. What this firewall
*is* genuinely good for: a contained lab segment (learning pf/pfSense without
risk), and — via Tailscale, not port-forwarding — reaching home from away.

## VM configuration (VMID 104)

| Setting | Value |
|---|---|
| Guest OS type | Other (FreeBSD), Netgate Installer ISO (`netgate-installer-v1.2-RELEASE-amd64.iso`), hash verified over the `.gz` **before** gunzip |
| Edition | CE at the installer prompt; `[A] Auto (ZFS)` install — snapshot/rollback inside the appliance |
| CPU / RAM | 2 cores (`host` type), 2 GB, ballooning 0 |
| Disk | 16 GB VirtIO SCSI on `local-lvm` |
| net0 | VirtIO on `vmbr0` — firewall WAN |
| net1 | VirtIO on `vmbr1` — firewall LAN (`<PFSENSE_LAN_IP>`, gateway of the segment) |
| Proxmox per-VM firewall | **OFF** — pve-firewall on top of a guest whose job *is* firewalling stacks a second drop layer and sends you debugging the wrong one |
| Boot | order `scsi0`, CD device cleared after install (leaving it re-enters the installer every boot) |

Host side of the segment: a port-less `vmbr1` bridge stanza written directly
into `/etc/network/interfaces` (creating it via `pvesh` with
`bridge_ports none` fails on this build), host bridge address `<PROXMOX_BRIDGE_LAN_IP>`.
A static-IP Alpine test LXC (VMID 105, `<LAB_TEST_LAN_IP>`) was created **before**
the firewall so the new segment would have something answering — without it,
the firewall's web UI can't be reached and nothing on the segment has an
address until the firewall is configured.

## Console + SSH channel

- Installer never asks for a password; the factory `admin` pair was replaced
  from console `3) Reset admin account and password`. Console `14) Enable
  sshd` opened the fast channel: from the hypervisor host,
  `sshpass -f /root/<credentials-file> ssh admin@<PFSENSE_LAN_IP> "<cmd>"` — a full
  query round trip in under a second, versus tens of seconds per console
  screenshot.
- Console menu `1)` (Assign Interfaces) and `2)` (Set IP) fix the classic
  install-time failure: the installer's network step only walks interfaces
  with an assigned role, so an unassigned NIC silently gets **no address**
  (post-install banner shows only the WAN line).
- `sshguard` runs on the appliance: repeated bad passwords **close** the
  session instead of re-prompting and can temporarily block the source — it
  looks exactly like a hang.

## Tailscale subnet router (reach home from away)

Tailscale is installed on the appliance itself, advertising the lab segment:

- Upstream `tailscale` pkg, authorized via login URL; auth state lives on the
  appliance and survives daemon restarts.
- pfSense does **not** boot-manage pkg-installed daemons — it regenerates
  `/etc/rc.conf` from its own config, so `sysrc ..._enable=YES` does not
  survive a reboot. Persistence is a small
  `/usr/local/etc/rc.d/tailscale-boot.sh` that starts `tailscaled` and runs
  `tailscale up --accept-dns=false --advertise-routes=<LAB_SEGMENT>`
  (`--accept-dns=false` so joining the tailnet can't hijack the resolver;
  the flag is `--advertise-routes`, not `--advertise-subnets`).
- Subnet-route advertisement is **half-finished** until two approvals land:
  the route approved for the device in the Tailscale admin console, **and**
  clients accepting routes (`tailscale set --accept-routes`; mobile clients
  pick routes up automatically). Until then `tailscale ping <lab-host>`
  answers `no matching peer` while the appliance's own tailnet IP pings
  fine — that signature points at the console, not the firewall.
- Subnet routers need **key expiry disabled** in the admin console or the
  node silently drops off when its key expires; read a node's expiry status
  from a *peer*, not from the node itself.
- **Gotcha:** `tailscale0` never appears in pfSense's *Available network
  ports* (that list holds only pfSense-managed ports), so it is deliberately
  **not assigned** as an interface — and doesn't need to be. The web GUI is
  reachable at `https://<PFSENSE_TAILNET_IP>` purely through pfSense's
  anti-lockout rule (http/https/ssh **to the firewall itself, on every
  interface**), with zero custom rules in the dump. Subnet-forwarding traffic
  is likewise handled without pfSense rules; admin-console approval is the
  gating step.

## DNS on the appliance

- DNS servers: Pi-hole (`<DOCKER_VM_LAN_IP>`) + `1.1.1.1`, with **DNS Server
  Override disabled** so WAN DHCP can never rewrite the list.
- The bundled unbound runs in **resolver (recursive) mode** (no
  `forward-zone` in its config) and answers lab clients with DNSSEC-validated
  results through WAN — the healthy default, not something to "fix". Pi-hole
  remains fleet-wide DNS for the tailnet; the firewall resolver is proven on
  the lab segment only. The test LXC points its nameserver at
  `<PFSENSE_LAN_IP>`, so lab clients resolve *through* the firewall.

## Verification playbook (what proved each layer)

```bash
# subnet router path, from the host and the desktop
tailscale ping <LAB_TEST_LAN_IP>        # pong ... via <PFSENSE_LAN_IP>:41641   (direct)
ping <LAB_TEST_LAN_IP>                  # 0% loss over the tailnet route (from a laptop)

# GUI over tailnet (anti-lockout proof, no custom rules needed)
curl -skI https://<PFSENSE_TAILNET_IP> | head -1   # HTTP/2 200

# appliance-side sanity
pfctl -s info                   # per-interface passed/blocked counters
pfctl -sr | grep -n <rule-id>   # effective ruleset, with rule positions
```

`/var/log/filter.log` on the WAN side fills with the home LAN's SSDP (1900)
and mDNS (5353) multicast dropped by the default-deny blocks — permanent
noise, not a misconfiguration. pfSense's default-deny blocks carry **no
interface qualifier**, so every interface the appliance gains later starts
inbound-blocked with no rule visible in the GUI.

## Snapshot hygiene

The guest keeps exactly one live Proxmox rollback snapshot; earlier milestone
snapshots were deleted after the next milestone verified. The thin pool has
little spare room and live snapshots pin real blocks — read pool consumption
(`lvs`, `vgs`) after each snapshot and prune stale ones. Deleting on this
Proxmox build: `qm delsnapshot <vmid> <name>`.

## Roadmap

- **Unified pfSense release 26.10** (Netgate merged CE and Plus into one
  edition, announced Oct 2026): free in-place upgrade from 2.9. Before it:
  snapshot. After it: reinstall the tailscale pkg and re-create the rc.d
  script — a major upgrade swaps the ZFS root and takes non-pfSense packages
  with it. Recommended before that upgrade: switch to `pfSense-pkg-Tailscale`
  from the Package Manager, the supported integration that adds a VPN →
  Tailscale page and a dedicated Tailscale rules tab.
- **Next lab exercises, in the order that actually works:** Packet Capture →
  Diagnostics → States → tick a row → "Add selected rule(s)" (pfSense writes
  the rule from traffic it already observed — do this before hand-writing
  rules) → Aliases → NAT → Outbound (why the ISP gateway never sees the
  segment's addresses) → DNS filtering → IDS. Target pattern: default-deny
  with one narrow exception.
- Promotion candidates: move unpatchable IoT onto its own segment; outbound
  WireGuard client for reachability analysis.
