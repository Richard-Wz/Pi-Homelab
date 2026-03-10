# Raspberry Pi 5 Homelab (Nextcloud + qBittorrent + Portainer + Tailscale)

This repo documents my Raspberry Pi 5 homelab build and provides the Docker Compose stack used to run:

- **Nextcloud** (with **MariaDB**)
- **qBittorrent**
- **Portainer**
- **Tailscale** for remote access (no public port-forwarding)

The design emphasizes **network isolation** (separate Docker networks) and a clean storage layout (named volumes + a bind-mounted downloads folder).

---

## Diagram

The main architecture diagram is in:

- `design_layout.svg`

It highlights:
- Clients (phone / laptop) using **Tailscale**
- Docker services and **network segmentation**
- Storage (named volumes + downloads bind mount)
- Nextcloud “External Storage” mount (read-only)

---

## Hardware / Software Assumptions

- Raspberry Pi 5
- NVMe HAT + NVMe SSD
- Raspberry Pi OS installed on NVMe
- Raspberry Pi Imager **v2.0.3**
- Docker + Docker Compose plugin

---

## 1) Change Raspberry Pi 5 Boot Priority (NVMe/USB Boot)

1. Plug in an SD card and open **Raspberry Pi Imager (v2.0.3)**
2. Under **Operating System**, choose:
   - `Misc utility images` → `Bootloader (Pi 5 family)` → `NVMe/USB Boot`
3. Write the bootloader image to the SD card
4. Insert the SD card into the Pi and boot once (let it apply/update)
5. Flash **Raspberry Pi OS** onto the NVMe drive (with Raspberry Pi Imager)
6. Install the NVMe into the HAT and boot the Pi from NVMe

---

## 2) Enable PCIe Gen 3 for NVMe (Pi 5)

Edit:

```bash
sudo nano /boot/firmware/config.txt
```

Add:

```ini
dtparam=pciex1
dtparam=pciex1_gen=3
```

Reboot:

```bash
sudo reboot
```

Optional performance test:

```bash
sudo apt update
sudo apt install hdparm -y
sudo hdparm -tT /dev/nvme0n1
```

---

## 3) Install Docker

Install Docker (official script):

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

Allow Docker without `sudo`:

```bash
sudo usermod -aG docker $USER
```

Disconnect SSH and reconnect, then verify:

```bash
groups
docker --version
docker compose version
```

---

## 4) Folder Setup

My working directory:

```bash
mkdir -p ~/docker/main/data/downloads
cd ~/docker/main
```

Copy `docker-compose.yaml` into `~/docker/main/`.

---

## 5) Create `.env` (Secrets)

Create a `.env` file **in the same directory** as `docker-compose.yaml`:

```bash
nano .env
```

Template:

```bash
MYSQL_ROOT_PASSWORD=<db_pwd>
NEXTCLOUD_DB_NAME=<nextcloud_username>
NEXTCLOUD_DB_USER=<nextcloud_username>
NEXTCLOUD_DB_PASSWORD=<db_pwd>
```

> `.env` contains secrets and must not be committed.

---

## 6) Start the Stack

From `~/docker/main`:

```bash
docker compose up -d --build
docker compose ps
```

---

## 7) Access (Tailscale IP + Ports)

Remote access method: **Tailscale IP + port**.

Replace `<ts-ip>` with your Pi’s Tailscale IP (`tailscale ip -4`).

- **Nextcloud:** `http://<ts-ip>:8080`
- **Portainer:** `http://<ts-ip>:9000` and `https://<ts-ip>:9443`
- **qBittorrent WebUI:** `http://<ts-ip>:8090`

---

## 8) Downloads Folder (qBittorrent → Nextcloud External Storage)

Shared folder on host:

- Host path: `~/docker/main/data/downloads`

Mounted into containers as:
- qBittorrent: `./data/downloads:/downloads` (rw)
- Nextcloud: `./data/downloads:/external/downloads:ro` (read-only)

To view downloads in Nextcloud:
1. Enable **External storage support** app
2. Add external storage:
   - Type: **Local**
   - Path: `/external/downloads`

This keeps Nextcloud from writing into the downloads folder (safer).

---

## Notes

- MariaDB is pinned to `mariadb:11.8` for Nextcloud compatibility (avoid `mariadb:latest`).
- `db_net` is an internal network so the database is not reachable from other containers.

---

## Optional Troubleshooting Commands

```bash
docker compose ps
docker compose logs nextcloud
docker compose logs nextcloud_db
docker compose logs qbittorrent
docker compose logs portainer
```

Validate shared downloads mount:

```bash
ls -la ~/docker/main/data/downloads
docker exec -it qbittorrent ls -la /downloads
docker exec -it nextcloud ls -la /external/downloads
```