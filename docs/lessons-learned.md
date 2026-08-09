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
