# Proxmox Firewall — Host Hardening + Per-Service Rules

The Proxmox firewall is configured with a **default-deny** posture: management
access is restricted to trusted networks, and each exposed service is allowed only
its specific port. This documents the config and — more usefully — how I debugged
it when a rule silently didn't work.

## Design

Proxmox firewall rules apply at three nested levels (Datacenter, Node, Guest) and
combine. The strategy:

- **Datacenter level:** allow management (web UI `8006`, SSH `22`) *only* from the
  LAN and the Tailscale network. Default input policy = **DROP**.
- **Guest (container) level:** the Minecraft container allows only TCP `25565`
  inbound and drops everything else.

Net result: the admin interfaces are never exposed to the public internet, and the
only internet-reachable port anywhere is the one the game actually needs.

## The golden rule: allow yourself in *before* enabling

A default-DROP firewall on a headless box will lock you out if you enable it before
adding an allow rule for your own access. Safe order:

1. Create the ACCEPT rules while the firewall is still **off** (nothing is enforced
   yet, so zero lockout risk).
2. Keep an SSH session open as a lifeline (`pve-firewall stop` instantly disables it).
3. Set default input policy to DROP.
4. Enable at the datacenter level, then the node level, checking access after each.

Proxmox also auto-generates a management allow-set (`PVEFW-0-management`) for the
LAN as a built-in safety net, but relying on explicit rules is cleaner.

## Management rules (Datacenter)

| Direction | Action | Proto | Port | Source | Purpose |
|-----------|--------|-------|------|--------|---------|
| in | ACCEPT | tcp | 8006 | `192.168.1.0/24` | Web UI from LAN |
| in | ACCEPT | tcp | 22 | `192.168.1.0/24` | SSH from LAN |
| in | ACCEPT | tcp | 8006 | `100.64.0.0/10` | Web UI from Tailscale |
| in | ACCEPT | tcp | 22 | `100.64.0.0/10` | SSH from Tailscale |

`100.64.0.0/10` is Tailscale's address range — allowing it preserves authenticated
remote admin without exposing anything publicly.

## Service rule (Minecraft container)

| Direction | Action | Proto | Port | Source |
|-----------|--------|-------|------|--------|
| in | ACCEPT | tcp | 25565 | any |

Source is `any` because remote players connect from arbitrary internet addresses.
Everything else inbound to the container is dropped by the default policy.

## Debugging: a rule that silently did nothing

After enabling the container firewall, remote players got **"Connection timed out"**
(not "refused"). That distinction mattered:

- **Refused** = reached the host, nothing listening (server down).
- **Timed out** = packets dropped in transit — the signature of a firewall.

So the firewall was the suspect. Rather than guess, I read what Proxmox actually
compiled to iptables:

```bash
pve-firewall compile
```

The container's inbound chain showed the problem — no rule for 25565:

```
veth100i0-IN
  -A veth100i0-IN -p udp --sport 67 --dport 68 -j ACCEPT   # DHCP only
  -A veth100i0-IN -j PVEFW-Drop                             # everything else dropped
  -A veth100i0-IN -j DROP
```

**Root cause:** the 25565 rule existed in the GUI with correct settings but was
**not enabled** (the per-rule checkbox was unchecked). In Proxmox, creating a rule
and *enabling* it are two separate steps — an unchecked rule compiles to nothing.
Several rules (including the management ones) were unchecked for the same reason.

After ticking Enable on each rule, the compiled chain showed the rule live:

```
veth100i0-IN
  -A veth100i0-IN -p udp --sport 67 --dport 68 -j ACCEPT
  -A veth100i0-IN -p tcp --dport 25565 -j ACCEPT           # now active
  -A veth100i0-IN -j PVEFW-Drop
  -A veth100i0-IN -j DROP
```

Remote players connected immediately after.

### Note on stateful return traffic
Proxmox handles connection tracking automatically — the compiled ruleset already
contains `-m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT`, so only the *new*
inbound connection to 25565 needed an explicit allow; the ongoing session's return
packets are accepted by the established-connection rule.

## Takeaways

- **`pve-firewall compile` is the source of truth** — debug by reading the compiled
  iptables chains, not the GUI's rule list.
- **A Proxmox rule must be created *and* enabled** — an unchecked rule is inert.
- **Timed out vs. refused** narrows a connectivity problem to firewall vs. service.
- **Least exposure:** the container accepts only its one required port; admin
  interfaces are reachable only from the LAN and Tailscale, never the public net.
