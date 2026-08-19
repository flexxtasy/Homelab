# Media Server — Jellyfin (LXC)

A dedicated LXC container for self-hosted media streaming — Jellyfin, with
Intel Quick Sync hardware transcoding. Added 2026-08-18/19. Immich (photo
backup) is planned for the same container.

## Why an LXC this time, not a VM

[Docker VM](docker-vm.md) deliberately runs Docker in a full VM to avoid
nesting containers-in-containers. This one breaks that pattern on purpose:
the ThinkCentre's iGPU (Intel UHD 630) needs to be passed through for
hardware transcoding, and giving a GPU to a full VM means the VM gets
*exclusive* access to it — the host loses it entirely. An LXC shares the
host kernel, so passing through just the render device node
(`/dev/dri/renderD128`) is simple and doesn't require giving up anything
else. Kept the container **unprivileged** (the secure default) — Proxmox
9's GUI has a proper "Add: Device Passthrough" feature that handles this
without needing a privileged container.

## Container specs

| Setting | Value |
|---|---|
| VMID | 102 |
| Name | Media |
| OS | Debian 12 (same template as docker-vm) |
| RAM | 4 GB, 1 GB swap |
| Cores | 2 |
| Root disk | 20 GB on `local-lvm` |
| Media storage | bind-mounted from a second physical drive (`<BACKUP_DRIVE_MOUNT>/media`), ~680GB free |
| GPU | `/dev/dri/renderD128` passed through (Intel Quick Sync) |
| Features | `nesting=1,keyctl=1` (Docker-in-LXC needs both) |

## Deployment

```yaml
services:
  jellyfin:
    container_name: jellyfin
    image: jellyfin/jellyfin:latest
    restart: unless-stopped
    volumes:
      - ./config:/config
      - ./cache:/cache
      - /media:/media
    devices:
      - /dev/dri/renderD128:/dev/dri/renderD128
    ports:
      - "127.0.0.1:8096:8096"
```

## Access

Tailscale only — installed the Tailscale client **inside the LXC itself**
(not just on the Proxmox host) and used `tailscale serve` the same way as
every other service: `https://media.<TAILNET_NAME>.ts.net`. Deliberately
different tradeoff from Vaultwarden/Homarr/Uptime Kuma though: this only
works because every device that needs Jellyfin (desktop, laptop, phone) is
already the same person's own devices on the same tailnet — a
household with a smart TV or other non-Tailscale devices would need LAN
access instead, since local network discovery doesn't cross into a tailnet.

## Backups exclude this drive

The [nightly Proxmox backup job](backups.md) covers every VM/LXC by
default, including this container's root disk — but its media *mount
point* is explicitly excluded (unchecked "Backup" on the mount point in
Proxmox). A movie library isn't the kind of data that needs a nightly
restore point, and including it would blow through the backup retention
budget for no benefit.

## Lessons learned

- **Typing a host path into Proxmox's "Add Mount Point" dialog does not
  create a bind mount.** It creates a brand-new, empty virtual disk on
  whatever storage pool is selected (defaults to the fast/small
  `local-lvm`, not the intended drive), and uses the typed path as the
  volume's *internal* mount location — not a link to that host directory
  at all. Confirmed by checking `pct config` after: `mp0:
  local-lvm:vm-102-disk-1,mp=/mnt/seagate/media,size=8G` — a whole new disk,
  not a bind. **Fix**: use `pct set <id> -mp0 <host-path>,mp=<container-path>`
  directly via CLI for an arbitrary host directory bind mount; the GUI
  field alone doesn't reliably express this for unprivileged containers.
- **Config changes to a running container's mount points and feature flags
  don't apply live** — `pct set` writes the config immediately, but the
  container needs a reboot before the change is real inside it. Caught this
  by testing (`touch` a file, check both sides) immediately after each
  change instead of assuming the `pct set` succeeding meant it worked.
- **Unprivileged LXCs remap root to an unprivileged host UID** (default
  range starts at 100000). A bind-mounted host directory still owned by the
  real host `root` is therefore unwritable from inside the container even
  though it "looks like" root owns it on both sides. Fixed with `chown -R
  100000:100000 <host-path>` to match the container's mapped root UID —
  confirmed with `/etc/subuid` rather than assuming the default mapping.
- **Docker-in-LXC needs `keyctl=1` in addition to `nesting=1`.** Only
  discovered this because it's the first LXC in the homelab running Docker
  (docker-vm is a full VM specifically to avoid this class of issue) —
  added proactively rather than waiting for an obscure failure.
- **Tailscale inside an unprivileged LXC needs `/dev/net/tun` passed
  through explicitly** — it's not there by default, unlike a full VM where
  the whole kernel device set is available. Symptom: `tailscaled` crash-loops
  with `CreateTUN("tailscale0") failed; /dev/net/tun does not exist`
  (found via `journalctl -u tailscaled`, not guesswork). Fix required two
  raw lines in the container's `/etc/pve/lxc/<id>.conf` (not exposed as a
  first-class `pct set` flag, unlike GPU passthrough):
  ```
  lxc.cgroup2.devices.allow: c 10:200 rwm
  lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
  ```
- **A brand-new Tailscale node's HTTPS cert isn't ready the instant
  `tailscale serve` is configured.** The very first request right after
  setup failed with a TLS-layer error (not a network/firewall error — TCP
  connected fine, the handshake itself failed). `journalctl -u tailscaled`
  showed the cert request completing (`cert(...): got cert`) within about
  a second of that failed request — a timing race on first use, not a real
  problem. Retrying moments later worked.
- **Jellyfin's setup wizard can mark itself "completed" without the admin
  account actually being created.** Symptom: every login attempt fails
  with "Invalid username or password" no matter what's entered, and
  "Forgot Password" appears to succeed (shows the "a file has been created"
  message) but no such file ever appears anywhere on disk. Root-caused by
  checking the actual data, not the UI's claims: `SELECT COUNT(*) FROM
  Users` against Jellyfin's SQLite database returned **0** — no account
  had ever been persisted — while `/config/config/system.xml` had
  `IsStartupWizardCompleted: true`. The "forgot password" flow was silently
  no-op'ing because there was no real user to reset (a reasonable
  anti-enumeration design, but a confusing symptom without checking the
  DB). **Fix**: stop the container, flip that flag back to `false` in
  `system.xml`, restart — Jellyfin re-shows the full setup wizard, and
  completing it properly (all the way to Finish) creates a real account.
  **Takeaway:** when a login is failing and a password-reset flow "succeeds"
  with no visible effect, check whether the account exists at all before
  assuming it's a credentials problem.
