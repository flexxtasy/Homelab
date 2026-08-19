# Proxmox Backups

Nightly automated backups of every VM/LXC to a dedicated storage drive —
added 2026-08-18 after realizing the homelab had **zero backups** despite
running for weeks.

## Setup

- **Target**: a second physical drive (`<BACKUP_DRIVE_DEVICE>`, ~700GB),
  mounted at `<BACKUP_DRIVE_MOUNT>`, registered as Proxmox storage `seagate`
  with `content backup` added (alongside its existing `images,vztmpl,iso`
  use for the same drive).
- **Retention**: `prune-backups keep-last=3` — keeps the 3 most recent
  backups per guest, older ones auto-pruned on each run.
- **Schedule**: `pvesh create /cluster/backup` — daily at 03:00, `--all 1`
  (every current *and future* VM/LXC, no need to update the job when adding
  new guests), `--mode snapshot` (no downtime), `--compress zstd`.

## Why snapshot mode + zstd

- **Snapshot mode**: backs up a live guest via an LVM-thin snapshot —
  no stop/suspend needed, matters since one of the backed-up guests runs
  Pi-hole (DNS for the whole network).
- **zstd compression + sparse detection**: a 20GB VM disk backed up to a
  2.7GB archive in the first real run — Proxmox detects zero-filled space
  and skips it, so actual usage is far below the nominal disk size.

## Verification

Don't just trust the schedule — ran a manual `vzdump <vmid list> --storage
seagate --mode snapshot --compress zstd` immediately after setup and
confirmed real archive files landed in the storage's `dump/` folder with
`ls -lh`, not just a "success" message.

## Lessons learned

- **Proxmox directory storages don't accept an arbitrary backup path.** A
  `dir` storage's content types (`images`, `vztmpl`, `iso`, `backup`, ...)
  each map to a *fixed* subfolder name under that storage's base path —
  backups always land in `<storage-path>/dump/`, not wherever you happen to
  `mkdir`. Manually creating a differently-named folder for backups first
  and expecting Proxmox to use it doesn't work — enable the `backup`
  content type on the storage and let Proxmox manage `dump/` itself.
- **`prune-backups keep-all=1` was already set on the storage by default**
  (a leftover from before `backup` content was even enabled) — worth
  checking explicitly, since "keep everything forever" silently becomes a
  real problem once backups start accumulating. Changed to `keep-last=3`.
