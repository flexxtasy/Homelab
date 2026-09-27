# Lessons Learned

Real problems hit while building this setup, and how I solved them. Documenting
troubleshooting is as valuable as documenting the working config — it shows the
debugging process, not just the end result.

## NixOS Workstation Migration

Migrated my gaming PC (Ryzen 7 8700F, RTX 5070 Ti) from CachyOS to NixOS with
Hyprland — a fully declarative, reproducible desktop.

### Nvidia on Blackwell (RTX 5070 Ti)
- 50-series (Blackwell) GPUs **require** the open kernel modules: `hardware.nvidia.open = true`.
- Symptoms of missing/wrong drivers: invisible/glitchy cursor, apps freezing on
  resize — all classic signs of software-rendering fallback.
- Verified working drivers with `nvidia-smi` (must show the GPU, driver, CUDA).

### CachyOS Kernel + Nvidia Version Lock
- Used `nix-cachyos-kernel` flake to get the CachyOS BORE kernel on NixOS.
- The BORE kernel (7.1.6) initially **failed to build the Nvidia module**:
  `fatal error: linux/of_gpio.h: No such file or directory`.
- Root cause: the bleeding-edge kernel removed a header the older Nvidia driver
  still referenced. Kernel and GPU driver must stay in sync.
- Fix: bumped the Nvidia driver to `nvidiaPackages.latest` (610.x), which supported
  the newer kernel. Alternative was the LTS kernel.
- **Takeaway:** with Nvidia, you run the newest kernel your driver supports, not the
  absolute newest kernel.

### Broken Binary Cache Key
- A leftover custom binary-cache public key poisoned all downloads
  (`error: public key is not valid`), even blocking unrelated rebuilds.
- Fix: removed the bad `nix.settings` substituter/key, and forced the official
  cache (`--option substituters "https://cache.nixos.org"`) to break the catch-22.
- **Takeaway:** a bad cache key in the running daemon config blocks everything until
  a successful rebuild replaces it — override on the command line to recover.

### Flakes Need `git`
- Flakes that fetch inputs from git repos require `git` present at evaluation time.
  A minimal NixOS install doesn't include it — installed it before those inputs
  could resolve.

## Minecraft Server (LXC)

Ran a vanilla 1.26.2 server in a Debian 12 LXC container on Proxmox.

### Java class-file version mismatch
- First launch crashed with `UnsupportedClassVersionError: class file version 69.0,
  this version only recognizes up to 65.0`.
- Decoded: class file 65.0 = Java 21, 69.0 = Java 25. The jar was compiled for a
  newer Java than was installed.
- Debian bookworm ships only Java 17 (backports didn't have 21 either), so I added
  the **Adoptium** repo and installed Temurin 25, then picked it with
  `update-alternatives --config java`.
- **Takeaway:** that error is a version map, not a mystery — the two numbers tell you
  exactly which Java you have vs. which you need.

### Whitelist verifies against Mojang
- `whitelist add <name>` does a live lookup against Mojang's API, so a mistyped
  username fails immediately with "That player does not exist" (plus a scary but
  harmless stack trace). The name must be the exact Java Edition username.

### "Connection refused" = nothing listening
- A remote client got `finishConnect() failed with error(-111): Connection refused`.
  That specific error means the packet reached the network but no process was
  listening on the port — i.e. the server wasn't running, not a firewall/forwarding
  problem. The cause: the server had been running in the foreground console, which
  ended when the console closed.
- **Fix + takeaway:** run the server in `screen` so it's decoupled from any console.
  "Connection refused" vs. "connection timed out" is a useful distinction — refused
  means reached-but-nothing-there, timed out usually means blocked/unreachable.

### LAN vs. public IP
- Same-network players use the container's **local** IP (`<CONTAINER_LAN_IP>`); remote
  players use the **public** IP + port-forward. Handing a LAN address to someone
  outside the network (or vice-versa) just fails to connect.
- The container IP was DHCP-**reserved** so the port-forward target can't drift.

## Proxmox Firewall

### A rule must be created *and* enabled
- Built the container's 25565 allow rule with all-correct settings, but remote
  players still timed out. The rule had an **unchecked enable box** — in Proxmox,
  creating a rule and enabling it are separate steps, and an unchecked rule compiles
  to nothing. Several rules were unchecked for the same reason.

### Debug by reading the compiled ruleset
- `pve-firewall compile` prints the actual iptables chains Proxmox generates. The
  container's inbound chain (`veth100i0-IN`) showed no 25565 line — proving the rule
  wasn't active regardless of how it looked in the GUI. Trust the compiled output,
  not the rule list.

### "Timed out" vs. "refused"
- Connection **refused** = reached the host, nothing listening (service down).
  Connection **timed out** = packets dropped in transit (firewall). This distinction
  pointed straight at the firewall as the cause rather than the server.

### Don't over-firewall a port that must be open
- The Minecraft port has to be internet-reachable — that's its job — so firewalling
  the container mainly protects its *other* ports (which aren't forwarded anyway).
  The real security win is at the host: management restricted to LAN + Tailscale with
  default-deny. Knowing where a control adds value (and where it's just complexity)
  matters as much as knowing how to configure it.

## Homepage Dashboard

See [Homepage](homepage.md) for full details.

### "Host validation failed" on first access
- Loaded fine via `curl localhost:3000` on the Docker VM itself, but the LAN
  IP and Tailscale hostname both errored in the browser and in `docker logs
  homepage` with `Host validation failed for: <host>:3000`.
- **Cause:** Homepage's Next.js base validates the `Host` header against an
  allowlist (DNS-rebinding protection) — anything not `localhost` needs to be
  explicitly allowed.
- **Fix:** add `HOMEPAGE_ALLOWED_HOSTS: <ip>:3000,<hostname>:3000,localhost:3000`
  under `environment:` in the compose file, then `docker compose up -d` to
  recreate the container.
- **Takeaway:** when a self-hosted app works from `localhost` on its own host
  but not from any other address, suspect host-header/origin validation before
  suspecting the firewall or DNS.

### Minecraft widget: vague browser error, real cause only in server logs
- Browser showed generic "API Error: Unexpected error" with zero detail.
  `docker logs homepage` had the real cause: `TypeError: Invalid URL`.
- **Cause:** Homepage's widget config docs show `url: <ip>:<port>` (no scheme)
  for the Minecraft widget, but its proxy layer calls `new URL()` on every
  widget's `url` regardless of type — a scheme-less string throws immediately.
- **Fix:** prefix with `http://` anyway (`url: http://<container-ip>:25565`)
  even though Minecraft isn't HTTP — Homepage still performs a raw
  server-list-ping under the hood, the scheme is only there to satisfy the URL parser.
- **Takeaway:** for any Homepage widget error that's vague in the browser,
  check `docker logs homepage` first — server-side errors are logged in full,
  client-side messages are generic by design.

### A tile's href silently pointed at the wrong service
- The "Docker VM" dashboard tile linked to the Docker VM's bare LAN IP (no
  port = 80), which happens to be Pi-hole's admin port on the same VM.
  Clicking it looked like a broken Pi-hole login rather than an unrelated link.
- **Takeaway:** on a multi-service host, a bare IP with no port is never
  "the host" — it's whatever happens to be listening on port 80. Don't link
  to a host generically; link to the actual service, or not at all.

### Scope API tokens to least privilege, even for read-only dashboards
- First pass created the Proxmox API token with `--privsep 0` (inherits root's
  full permissions) just to get the dashboard widget working fast.
- Redone properly: `--privsep 1` + explicit `PVEAuditor` (read-only) ACL grant
  at path `/`. A dashboard only ever needs to *read* status.
- **Takeaway:** "just get it working" credentials have a way of becoming
  permanent — worth the extra two commands to scope it correctly the first time.

### Full homelab security audit found the dashboard was the real hole (2026-08-18)
- Ran a proper audit across every homelab host (config review + `nmap`
  service/port scans + live SSH-auth-method probes) rather than assuming
  things were fine. Most of the homelab checked out clean — Proxmox's full
  65535-port surface was just SSH + the web UI, Vaultwarden and the AI
  assistant were correctly Tailscale-only, no privileged Docker containers,
  no secrets in the AI harness code.
- **The one real finding: Homepage itself.** It was bound to `0.0.0.0:3000`
  (reachable by *any* device on the LAN, not just trusted ones) with zero
  built-in authentication — and its `services.yaml` held the Proxmox API
  token and the Pi-hole widget key in plaintext, both fetchable by loading
  the page. The Pi-hole key turned out to double as a reused personal login
  password, which is the actual worst part — a LAN-exposed dashboard
  silently handing out a password used elsewhere.
- **Mitigating factor, confirmed rather than assumed:** the Proxmox token
  was already `--privsep 1` + `PVEAuditor` (read-only) from the earlier
  lesson above, so the exposure was read-only recon, not a way to modify or
  destroy anything. Verified this directly (`pveum acl list`) instead of
  trusting the "should have been scoped" assumption.
- **Fix:** same pattern already used for [Vaultwarden](vaultwarden.md) and
  the AI assistant — moved off the LAN entirely instead of trying to bolt on
  auth. Compose file changed to `ports: "127.0.0.1:3000:3000"`, then
  `tailscale serve --https=8443 http://127.0.0.1:3000` (a second `serve`
  port on the same Tailscale node, since 443 was already Vaultwarden's).
  Also rotated the exposed credentials: the old `root@pam!homepage` token
  was deleted outright and replaced with a dedicated `homepage@pve` user
  (still `--privsep 1` + `PVEAuditor` only — least-privilege *and* not
  attached to `root@pam` at all now), and the Pi-hole widget was switched
  from the reused admin password to a dedicated API app password.
- **Takeaway:** "LAN-only" is not the same as "safe" — anything bound to
  `0.0.0.0` is reachable by every device on the network, including ones you
  don't fully trust (guests, IoT, anything compromised). A dashboard that
  aggregates credentials for *other* services is a higher-value target than
  any one of those services alone, and deserves the strictest access model
  in the homelab, not the loosest. When in doubt, default to Tailscale-only
  like everything else here, rather than LAN-open-with-a-plan-to-add-auth-later.
- **Same audit also found SSH password login still enabled on both the
  Proxmox host and the Docker VM** (`PermitRootLogin yes` on Proxmox,
  meaning root itself was password-reachable over SSH from anywhere on the
  LAN) — confirmed with a live auth-method probe against each host, not just
  by reading config. Both already had working key-based access, so this was
  pure unnecessary exposure. Fixed by dropping
  `PasswordAuthentication no` (+ `PermitRootLogin prohibit-password` on
  Proxmox) into `/etc/ssh/sshd_config.d/99-harden.conf` on each host — a
  drop-in file rather than editing the main config, so it's an isolated,
  easily-reverted override — then `sshd -t` to validate syntax before
  restarting, and re-probed both afterward to confirm only `publickey` is
  offered now. **Takeaway:** if key-based access already works, password
  auth is a door you don't use but still have to defend — turn it off.

## Media Stack (Radarr, Sonarr, Prowlarr, qBittorrent, FlareSolverr)

See [Media Stack](media-stack.md) for full details — Cloudflare-bypass
setup, a real malware-torrent incident, and a three-layer qBittorrent auth
debugging saga. Highlights:

- Prowlarr assigns a proxy (FlareSolverr) to an indexer via matching
  **Tags** on both sides, not a "Proxy" dropdown.
- Prowlarr's Radarr/Sonarr sync can silently no-op — use
  `POST /api/v1/command {"name":"ApplicationIndexerSync","forceSync":true}`.
- A `docker cp`'d SQLite file can lag behind the live app due to WAL —
  query the running app's API instead of trusting a raw file copy.
- **Auto-grab with no review pulled down real malware** (a `.exe`
  disguised as a movie release) from a public tracker. Fixed by disabling
  RSS/automatic search per-indexer while leaving interactive (manual)
  search on — nothing grabs without a human looking at it first.
- qBittorrent's WebUI had three stacked auth bugs once moved behind
  Tailscale: its temp password only works from true loopback, its
  host-header validation rejects any reverse-proxied/port-remapped
  request, and its brute-force IP ban lives only in memory (clears on
  restart, and doesn't block true-localhost access even while an
  externally-triggered ban is active).
- A password already flagged as compromised (from the Pi-hole incident)
  got typed into chat again as a lightly-modified variant — caught and
  replaced with something unrelated rather than trusting a "rotated"
  password that's really the same secret with different punctuation.

## Vaultwarden

See [Vaultwarden](vaultwarden.md) for full details.

### `tailscale serve` beats running your own reverse proxy for tailnet-only services
- Needed valid HTTPS for Vaultwarden (self-signed certs train you to click
  through browser warnings — bad habit). Normally that means Caddy/nginx +
  manually renewed certs.
- `tailscale serve --bg <port>` gets a real, auto-renewing cert from
  Tailscale's CA for the tailnet hostname, and only accepts connections from
  other tailnet devices — no extra container, no cert files to manage, no
  port to accidentally expose wider than intended.
- **Takeaway:** for anything that only needs to be reachable within the
  tailnet (not the LAN, not the internet), `tailscale serve` is simpler and
  arguably more secure than a self-managed reverse proxy.

### Two one-time settings block `tailscale cert`/`serve`, both easy to miss
- `tailscale cert` failed with `Access denied` until running
  `sudo tailscale set --operator=<user>` once (lets that user control
  tailscaled without sudo every time).
- Separately failed with `"your Tailscale account does not support getting
  TLS certs"` until enabling **HTTPS Certificates** in the tailnet's admin
  console (`https://login.tailscale.com/admin/dns`) — off by default per-tailnet.
- **Takeaway:** both are one-time, low-risk settings, but neither has an
  obvious error pointing at "go flip this setting in the admin console."

### Escaping `$` in Docker Compose env values
- An Argon2id hash (`$argon2id$v=19$...`) placed directly in a compose file's
  `environment:` value gets silently mangled — Compose treats `$VAR`/`${VAR}`
  as variable substitution.
- **Fix:** escape every `$` as `$$` in the YAML.
- **Takeaway:** any secret containing `$` (hashes, some generated passwords)
  needs this escape in Compose — worth checking for `$` before pasting any
  generated value into a compose file.

### Keep the admin credential as a hash, not plaintext
- Generated the Vaultwarden `ADMIN_TOKEN` as a random secret, then stored only
  its Argon2id hash in the compose file (`argon2` CLI, Debian package). The
  plaintext token was shown once and never written to any file on the server.
- **Takeaway:** for any "admin panel gated by one shared secret" pattern,
  default to hashing it the same way you'd hash a real password — a compose
  file being read by something/someone it shouldn't doesn't hand over the
  actual credential.

## Proxmox Host — NIC Hang Took Down the Whole Homelab (2026-08-25)

Woke up to the entire homelab unreachable — not one service, everything at
once. Traced it all the way back rather than just restarting things and
moving on.

### Root cause: onboard NIC hardware hang, not a software problem
- `journalctl -b -1` on the Proxmox host showed the real story: at 07:00:30,
  the kernel logged `e1000e: Detected Hardware Unit Hang` on the onboard
  NIC, then **repeated every ~2 seconds continuously for nearly two hours**
  — the driver's own watchdog kept trying to reset the interface and
  immediately hanging again, never recovering on its own. Since this is
  the host's *only* network path (LAN, Tailscale, and every VM/LXC's
  bridged networking all ride the same physical NIC), losing it took the
  whole stack down at once — nothing was actually broken inside any VM or
  container, they just had no network to be reached on.
- Checked for a triggering event at that exact timestamp (other log
  activity, the nightly backup job) — nothing correlated. Backup runs at
  03:00, four hours earlier; ruled out.
- **Real cause: Energy Efficient Ethernet (EEE) was enabled** on the NIC —
  `ethtool --show-eee` confirmed it active. This specific failure
  signature (repeating hang, stuck TX descriptor) is a well-documented
  e1000e/EEE interaction issue, not a one-off fluke.

### Fix
- Disabled EEE immediately: `ethtool --set-eee <nic> eee off`.
- Made it persistent — a runtime `ethtool` change alone reverts on the
  next boot. Added a `post-up` hook to the physical interface's stanza in
  `/etc/network/interfaces`:
  ```
  iface <nic> inet manual
  	post-up /usr/sbin/ethtool --set-eee <nic> eee off || true
  ```
  Verified the file still parses (`ifquery <nic>`) before trusting it.

### Recovery took longer than it should have — separate, compounding issue
- **None of the guest VMs/LXCs had `onboot` set** in Proxmox. After the
  host came back up, every VM/LXC had to be started by hand — the actual
  outage was ~2 hours, but full service recovery took longer because
  nothing came back automatically.
- Fixed: `qm set <vmid> --onboot 1` / `pct set <vmid> --onboot 1` on the
  always-on guests (the Docker VM, the media LXC). Deliberately **left
  the on-demand LXC (Minecraft) without onboot** — it's meant to be
  started manually when actually in use, not always-running.

### Takeaways
- **"Everything is down at once" almost always means one shared
  dependency, not N simultaneous failures** — check the host/network
  layer first, not each service individually. Going straight to
  per-service checks on a total outage is looking in the wrong place.
- `journalctl -b -1` (the *previous* boot's log) is how you find out what
  actually happened before a crash/reboot — the current boot's log starts
  clean and won't show you the cause.
- EEE is a common, low-risk-to-disable culprit for unexplained e1000e NIC
  hangs — worth checking `ethtool --show-eee` early on any "one NIC just
  stopped working" symptom before assuming failing hardware.
- `onboot` defaults to off in Proxmox — worth setting explicitly on
  anything meant to be always-available, or a host-level fault turns into
  a much longer outage than the fault itself caused.

## Update Automation & Least Privilege (2026-09-24)

### Security-only unattended-upgrades needs `#clear`
Debian's default `50unattended-upgrades` already lists several origins. Adding a
local file that only *appends* the security origin changes nothing. The override has
to reset the list first:

```
// /etc/apt/apt.conf.d/52unattended-upgrades-local
#clear Unattended-Upgrade::Origins-Pattern;
Unattended-Upgrade::Origins-Pattern {
    "origin=Debian,codename=${distro_codename}-security,label=Debian-Security";
};
Unattended-Upgrade::Automatic-Reboot "false";
```

Verify with `unattended-upgrade --dry-run -d | grep "Allowed origins"`. On the Proxmox
host this means Debian security fixes install nightly, while the Proxmox kernel,
`pve-manager`, QEMU and LXC packages stay manual.
- **Takeaway:** auto-install what's low-risk and time-critical (security patches);
  keep what can break the hypervisor behind a human decision. Always dry-run to see
  the origins actually in effect.

### Container updates: pre-download nightly, apply on approval
A nightly script runs `docker pull` for every running container's image and compares
the new image ID to the running one. Nothing restarts. The morning report lists which
apps have a new image ready, and applying them is one approved `compose up -d` per
project, with `docker image prune -f` afterwards. On a 19 GB VM disk the pulls alone
moved usage from 46% to 65%, and the prune reclaimed 3.8 GB.
- **Takeaway:** splitting "download" from "restart" gives you auto-update convenience
  without surprise restarts, but budget disk space for two copies of every image.

### Monitoring accounts drift into full root
The desktop watcher's SSH account on the Proxmox host had `NOPASSWD: ALL`, left over
from an earlier AI-agent experiment. Its sudo log showed it only ever ran five
read-only commands. Replaced with a root-owned wrapper with fixed subcommands
(`homelab-status lxc-list`, `jellyfin-state`, …) and a sudoers line allowing only that
script. Tested that `sudo id` and `sudo pct` are refused and the watcher still passes.
- **Takeaway:** check `journalctl _COMM=sudo` for what an automation account really
  uses, then grant exactly that. A monitoring key with root is a hypervisor key.

### Uptime Kuma had monitors but nobody to tell
Six monitors, zero notification channels, so outages were recorded but never sent
anywhere. Added a Discord notification applied to all monitors, plus `maxretries 2`
so a single failed check doesn't page.
- **Takeaway:** test the alert path end to end (the notification's *Test* button),
  not just that the monitor turns red.

### Minecraft graceful stop under systemd
Two bugs before `systemctl restart minecraft` saved the world and stopped quickly:
1. `pgrep -f "java .*server.jar"` inside `sh -c '…'` matched the shell itself (its own
   command line contains the pattern), so the wait loop never ended and systemd
   killed it at the 120 s timeout. Fixed by waiting on `$MAINPID` instead.
2. `screen -X stuff "stop$(printf '\r')"` never pressed Enter: the server received
   `stopr` (systemd's own escape handling ate a backslash), and later a raw CR still
   didn't submit the line. Screen's own escape, `stuff 'stop\015'` in single quotes
   inside a small script, works. The leftover `stopr…` text sat in the console until a
   real Enter arrived and was rejected as an unknown command. Harmless, but a reminder
   to check the log, not just the exit code.
- **Takeaway:** keep non-trivial `ExecStop` logic in a script file (no systemd escaping
  layer), and prove a graceful stop by reading the server log for the save message.

### A single DNS server for the whole tailnet
Tailscale's global nameserver is the Pi-hole container, with no second resolver.
Rebooting the Docker VM (or the Proxmox host under it) takes internet name resolution
away from every tailnet device for a few minutes.
- **Takeaway:** add a secondary public resolver in the Tailscale admin console, and
  schedule host reboots knowing DNS goes with them.

### Docker's embedded DNS needs more than one upstream
The media containers resolved names through a single upstream (the tailnet's DNS resolver).
Whenever that one resolver had a brief hiccup, every indexer went "unavailable" at once —
because Docker's embedded resolver had nothing to fall back to. Fixed by giving the Docker
daemon an ordered DNS list in `/etc/docker/daemon.json`, then restarting Docker:

```json
{ "dns": ["<TAILNET_DNS_IP>", "<PIHOLE_LAN_IP>", "1.1.1.1"] }
```

Containers pick it up as the `ExtServers` list behind `127.0.0.11`.
- **Takeaway:** Docker's embedded DNS only uses the daemon's `dns` list — give it several
  resolvers (tailnet, LAN Pi-hole, a public fallback) so one upstream blip doesn't knock out
  name resolution for every container.

### A service rooted in an account you want to remove
The *arr media stack's compose file and app-data lived under a **decommissioned service
account's** home directory (left over from an earlier setup). That made the account impossible
to lock/clean without breaking the stack. Moved the compose file and `app-data/` into the
primary user's home, `docker compose down` in the old path, `up -d` in the new one, then verified
each app's own health endpoint. UID/GID (`PUID/PGID`) stayed the same so file ownership on the
media share didn't change.
- **Takeaway:** don't leave a running service rooted in an account you plan to retire — the
  dependency is invisible until the cleanup fails. Keep service data under an account that stays.

### Local AI assistant: read freely, act only behind a human-confirmed code
A small local LLM (running on the workstation, reaching the homelab over the tailnet) does two
jobs: a **morning status report + outage alerts** (deterministic SSH checks, phrased by the model —
the facts are gathered by code, only the wording is the model's), and **approval-gated maintenance**
(restart a container, apply pending updates, reboot a host). Design rules that made it safe:
- **Read-only status needs no approval; anything that changes state needs a one-time code** sent
  out-of-band (a chat DM the model can't read), so the model can *propose* an action but can never
  approve its own.
- The agent has a **fixed allowlist of actions**, never a shell; on each host its SSH key is a
  forced-command key that only runs a specific dispatcher script.
- The monitoring account is **read-only** (a root-owned wrapper of fixed sub-commands), not general sudo.
- **Takeaway:** an LLM is great at *reading* infrastructure and drafting changes, but it should only
  *act* through a fixed action list, behind a human-confirmed token delivered out-of-band — never by
  handing it a shell or letting it approve itself.

## General Linux / Storage

### Partition != Filesystem
- `fdisk` creates a partition table entry, but the partition has no filesystem until
  `mkfs` is run. A skipped `mkfs.ext4` caused a mount failure with no obvious cause.

### fstab UUID Gotchas
- Use the filesystem `UUID` from `blkid`, not the `PARTUUID`.
- No quotes, no angle brackets around the UUID in `/etc/fstab`.
- Run `systemctl daemon-reload` after editing fstab on systemd systems.

## Recovery / Safety

- NixOS boot generations = instant rollback. Every rebuild is a new generation; a
  broken config never bricks the system (it fails to build, leaving the running
  system untouched, or you boot the previous generation).
- Always confirm a block device (`lsblk`) before any destructive operation
  (format, `dd`, wipe). Unplugging drives you're *not* targeting removes the risk
  of selecting the wrong one.
