# Docker VM

A dedicated VM on Proxmox for running Docker containers. Runs Docker in a VM
(not an LXC) for cleaner isolation and to match real-world / production practice.

## Why a VM instead of an LXC

Docker uses containerization itself, so running it inside an LXC means nesting
containers inside a container. That works but requires relaxing security
(`nesting=1`, sometimes privileged containers), and hits odd storage/networking
edge cases. A dedicated VM keeps Docker's security assumptions intact, behaves
exactly like the official docs describe, and is the standard architecture.

## VM specs

| Setting | Value |
|---|---|
| VMID | 101 |
| Name | docker (hostname `docker-vm`) |
| OS | Debian 12 (netinst) |
| RAM | 2 GB |
| Cores | 2 |
| Disk | 20 GB on `local-lvm` (NVMe) |
| IP | `<DOCKER_VM_LAN_IP>` (DHCP reserved) |
| Interface | `ens18` |

## Debian install notes

- Minimal netinst install; in **Software selection**, unchecked the desktop
  environment and checked only **SSH server** + **standard system utilities**
  (headless server, no GUI).
- Guided partitioning, entire disk, all files in one partition.
- Because a root password was set during install, Debian did **not** install
  `sudo` or add the user to the sudo group. Fixed post-install:

```bash
su -
apt-get update && apt-get install -y sudo
usermod -aG sudo <user>
exit
# log out / back in for group change to take effect
```

## Installing Docker (official repo)

Used Docker's official repository (not `apt install docker.io`) to get current
Docker Engine + Compose plugin.

```bash
# Prerequisites
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg

# Docker's GPG key
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Docker repo
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Verify
sudo docker run hello-world
```

## Run docker without sudo

```bash
sudo usermod -aG docker $USER
# log out / back in, then:
docker ps   # should work without sudo
```

## Resource check before creating the VM

Confirmed the host had headroom so the Minecraft LXC wouldn't be starved:

```bash
free -h    # showed ~9.5 GB available
pct list   # only the Minecraft container (VMID 100) was running
```

A 2 GB Docker VM alongside the 6 GB Minecraft LXC fits comfortably on 16 GB.

## Lessons learned

- Debian minimal install skips `sudo` when a root password is set — expected, not a bug.
- Use Docker's official repo, not the distro's `docker.io`, for a current version.
- Check host RAM (`free -h`) **before** allocating a new VM — good habit.
