# Project Plan: Home Media Server

## Overview

Set up an old laptop as a headless media server running Jellyfin, with automated torrent downloading via qBittorrent and Sonarr/Radarr. Any device on your home WiFi will be able to stream from it.

**Estimated total time:** 1–2 weekends  
**Difficulty:** Beginner–Intermediate

---

## Phase 1 — Prepare the Laptop

- [ ] Back up any files you want to keep from the laptop
- [ ] Download **Ubuntu Server 24.04 LTS** ISO from ubuntu.com
- [ ] Flash the ISO to a USB stick using **Balena Etcher**
- [ ] Boot the laptop from USB and run the installer
  - Set hostname to something like `mediaserver`
  - Enable OpenSSH Server when prompted
  - Create a user account (e.g. `media`)
- [ ] Complete installation and reboot
- [ ] SSH into the machine from your main computer to confirm it works:
  ```bash
  ssh media@<laptop-ip>
  ```
- [ ] Log into your **router admin panel** and assign a static/reserved IP to the laptop's MAC address (e.g. `192.168.1.100`)
- [ ] Update the system:
  ```bash
  sudo apt update && sudo apt upgrade -y
  ```

---

## Phase 2 — Storage

- [ ] Plug in an external USB drive (1TB+ recommended)
- [ ] Identify the drive:
  ```bash
  lsblk
  ```
- [ ] Format it as ext4 (replace `sdX` with your drive identifier — **double check this**):
  ```bash
  sudo mkfs.ext4 /dev/sdX1
  ```
- [ ] Create mount point and mount the drive:
  ```bash
  sudo mkdir -p /mnt/media
  sudo mount /dev/sdX1 /mnt/media
  ```
- [ ] Make it mount automatically on reboot by adding to `/etc/fstab`:
  ```bash
  sudo blkid /dev/sdX1  # Note the UUID
  sudo nano /etc/fstab
  # Add line: UUID=<your-uuid> /mnt/media ext4 defaults 0 2
  ```
- [ ] Create your folder structure:
  ```bash
  sudo mkdir -p /mnt/media/{films,tv,downloads}
  sudo chown -R media:media /mnt/media
  ```

---

## Phase 3 — Install Jellyfin

- [ ] Add the Jellyfin apt repository and install:
  ```bash
  curl https://repo.jellyfin.org/install-debuntu.sh | sudo bash
  ```
- [ ] Enable and start the service:
  ```bash
  sudo systemctl enable --now jellyfin
  ```
- [ ] Open the web UI from another device on your network:
  ```
  http://192.168.1.100:8096
  ```
- [ ] Complete the setup wizard:
  - Create an admin account
  - Add your media libraries pointing to `/mnt/media/films` and `/mnt/media/tv`
- [ ] Install the **Jellyfin app** on your TV, phone, or tablet and confirm streaming works

---

## Phase 4 — Install qBittorrent

- [ ] Install qBittorrent headless:
  ```bash
  sudo apt install qbittorrent-nox -y
  ```
- [ ] Enable and start the service:
  ```bash
  sudo systemctl enable --now qbittorrent-nox@media
  ```
- [ ] Access the web UI:
  ```
  http://192.168.1.100:8080
  ```
  Default credentials: `admin` / `adminadmin` — **change these immediately**
- [ ] In settings, set the default download path to `/mnt/media/downloads`

---

## Phase 5 — Automate with Sonarr & Radarr (Optional but Recommended)

Sonarr handles TV shows; Radarr handles films. Both monitor for new content and automatically trigger downloads.

- [ ] Install Sonarr:
  ```bash
  # Follow official Sonarr Linux install guide at sonarr.tv
  sudo systemctl enable --now sonarr
  ```
- [ ] Access Sonarr at `http://192.168.1.100:8989` and configure:
  - Connect it to qBittorrent (Settings → Download Clients)
  - Set root folder to `/mnt/media/tv`
  - Add indexers (torrent sites/RSS feeds)
- [ ] Repeat for Radarr at `http://192.168.1.100:7878` with root folder `/mnt/media/films`
- [ ] Test the pipeline by adding a show in Sonarr and confirming it downloads and appears in Jellyfin

---

## Phase 6 — VPN (Recommended)

Running a VPN on the torrent client prevents your ISP seeing torrent traffic.

- [ ] Sign up for **Mullvad** VPN (no account name required, privacy-focused)
- [ ] Install Mullvad on the server and configure it to only apply to qBittorrent traffic using split tunnelling, or use a WireGuard config bound to qBittorrent specifically
- [ ] Confirm your IP is masked before downloading anything

---

## Phase 7 — Remote Access (Optional)

To access your server from outside your home network:

- [ ] Set up **Tailscale** (free, dead simple VPN mesh):
  ```bash
  curl -fsSL https://tailscale.com/install.sh | sh
  sudo tailscale up
  ```
- [ ] Install Tailscale on your phone/laptop and your server will be accessible from anywhere via its Tailscale IP

---

## Maintenance Notes

- **Drive space:** Monitor with `df -h /mnt/media` — set up alerts if you want
- **Updates:** Run `sudo apt update && sudo apt upgrade` monthly
- **Jellyfin metadata:** It fetches artwork and descriptions automatically; if anything looks wrong, trigger a library rescan from the dashboard
- **Backups:** The media itself is replaceable; back up your Sonarr/Radarr/Jellyfin config folders if you want to preserve your library settings

---

## Useful Ports Reference

| Service | Port |
|---|---|
| Jellyfin | 8096 |
| qBittorrent | 8080 |
| Sonarr | 8989 |
| Radarr | 7878 |
