# Minecraft Server (Vanilla 1.26.2)

Self-hosted vanilla Minecraft: Java Edition server running in an LXC container on
the Proxmox homelab. Used by family on the LAN and friends over the internet.

## Architecture

- **Platform:** Debian 12 LXC container on Proxmox (unprivileged)
- **Resources:** 4 vCPU, 6 GB RAM, 20 GB disk (on NVMe / `local-lvm` for fast chunk I/O)
- **Java:** Eclipse Temurin 25 (Adoptium)
- **Server jar:** official Mojang vanilla server for 1.26.2
- **Process management:** runs inside a `screen` session so it survives console close

## Why LXC (not a VM)

A container is far lighter than a full VM for a single service — no guest kernel or
virtualized hardware overhead — so more of the host's resources go to the game.

## The Java version gotcha

Minecraft 1.26.2 requires **Java 25**. The first launch failed with:

```
UnsupportedClassVersionError: ... class file version 69.0,
this version of the Java Runtime only recognizes class file versions up to 65.0
```

Class file version 65.0 = Java 21, 69.0 = Java 25 — so the jar was built for a newer
Java than was installed. Debian 12's repos (and even backports) only ship Java 17/21,
so the fix was to add the **Adoptium** repo and install Temurin 25, then select it
with `update-alternatives --config java`.

## Setup

```bash
# In the container:
apt update && apt upgrade -y

# Java 25 via Adoptium (Debian only ships older Java)
apt install wget apt-transport-https gnupg screen -y
mkdir -p /etc/apt/keyrings
wget -qO - https://packages.adoptium.net/artifactory/api/gpg/key/public \
  | gpg --dearmor | tee /etc/apt/keyrings/adoptium.gpg > /dev/null
echo "deb [signed-by=/etc/apt/keyrings/adoptium.gpg] https://packages.adoptium.net/artifactory/deb bookworm main" \
  > /etc/apt/sources.list.d/adoptium.list
apt update && apt install temurin-25-jre -y
update-alternatives --config java   # select the Temurin 25 entry
java -version                        # confirm 25

# Server files
mkdir -p /opt/minecraft && cd /opt/minecraft
wget -O server.jar "<version-specific mojang server.jar URL from minecraft.net/download/server>"

# First run generates eula.txt, then stops
java -Xmx6G -Xms2G -jar server.jar nogui

# Accept EULA and start for real
sed -i 's/eula=false/eula=true/' eula.txt
```

## Running it persistently (screen)

The server must not be tied to the Proxmox web console — closing the console would
kill it. It runs inside a detachable `screen` session instead:

```bash
cd /opt/minecraft
screen -S minecraft
java -Xmx6G -Xms2G -jar server.jar nogui   # wait for: Done! For help, type "help"
# detach, leaving the server running in the background:  Ctrl+A then D
```

Manage it later:
```bash
screen -r minecraft     # re-attach to type server commands
# Ctrl+A then D to detach again
```

### Auto-start: systemd service wrapping screen

Manual `screen` doesn't survive a container or host reboot, and the LXC itself had no
`onboot` set — so an unattended Proxmox reboot would have left the server off. The fix
keeps the `screen` console but lets systemd own it:

```ini
# /etc/systemd/system/minecraft.service
[Service]
WorkingDirectory=/opt/minecraft
ExecStart=/usr/bin/screen -DmS minecraft /usr/bin/java -Xmx6G -Xms2G -jar server.jar nogui
ExecStop=/usr/local/bin/minecraft-stop
TimeoutStopSec=120
Restart=on-failure
```

```sh
# /usr/local/bin/minecraft-stop — type "stop" into the console so the world saves,
# then wait for the service's main process to exit
screen -S minecraft -p 0 -X stuff 'stop\015'
while [ -n "${MAINPID:-}" ] && kill -0 "$MAINPID" 2>/dev/null; do sleep 1; done
```

Plus `pct set <CTID> --onboot 1` on the Proxmox host. A graceful restart now takes ~1 s
of stop time and the log shows `All dimensions are saved` before exit. See
[lessons learned](lessons-learned.md#minecraft-graceful-stop-under-systemd) for the two bugs
it took to get there.

## Access

The container IP is DHCP-**reserved** at `<CONTAINER_LAN_IP>` so the address never changes
(otherwise the port-forward would eventually break).

| Who | Where | Address |
|-----|-------|---------|
| Same-house (LAN) | e.g. my brother | `<CONTAINER_LAN_IP>:25565` |
| Remote friends | their own homes | `<YOUR_PUBLIC_IP>:25565` (public IP) |

Remote access works via a router **port-forward** of TCP `25565` -> `<CONTAINER_LAN_IP>`
(set up via the ISP's router-management app; external and internal port both 25565). Friends need
no extra software — just the public IP.

### Keeping a public port friends-only
Port 25565 is exposed to the internet, so two safeguards keep it private:
- `online-mode=true` — only authenticated (legitimate) Minecraft accounts can join.
- **Whitelist enabled** — only explicitly added usernames can connect.

```
# in the server console (screen -r minecraft)
whitelist on
whitelist add <exact-username>
```
Note: `whitelist add` verifies the name against Mojang's servers, so it must be the
player's **exact** Java Edition username — a typo returns "That player does not exist."

```
# server.properties
white-list=true
online-mode=true
max-players=10
```

## Status

- [x] Java 25 installed (Adoptium Temurin)
- [x] Server running 1.26.2, world generated
- [x] Running under `screen` (survives console close) — confirmed, players joined
- [x] Whitelist enabled + members added
- [x] IP reserved (<CONTAINER_LAN_IP>) + port-forward (25565)
- [x] **systemd service** + LXC `onboot` for auto-start on container/host reboot
- [ ] Scheduled world backups to the Seagate storage

## Caveat: dynamic public IP

The public IP (`<YOUR_PUBLIC_IP>`) is dynamic and can change on the ISP's schedule.
If it changes, remote friends need the new address. A future improvement is Dynamic
DNS (DDNS) to map a stable hostname to whatever the current IP is.

## Capacity

6 GB allocated comfortably supports ~8-10 players on vanilla.
