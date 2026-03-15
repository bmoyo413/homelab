# 🏠 Homelab

![GitHub last commit](https://img.shields.io/github/last-commit/bmoyo413/homelab)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

Personal homelab running on two Proxmox hosts. Services run across Docker containers, LXCs (provisioned via Proxmox Helper Scripts), and dedicated VMs.

---

## 🖥️ Hardware

| Host | CPU | GPU | RAM | Storage |
|------|-----|-----|-----|---------|
| `pve-01` | Ryzen 5 2600X | RX 580 | 32GB DDR4 | 4TB Mirror (2x 4TB) |
| `pve-02` | Ryzen 5 2600 | GTX 1660 (passthrough) | 24GB | ZFS RAIDZ2 (6x 1TB) + 3x 256GB SSD Stripe |

**`pve-02` storage layout:**
- RAIDZ2 (6x 1TB) — media storage
- SSD Stripe (3x 256GB) — appdata / Docker volumes

---

## 🗂️ Repo Structure
```
homelab/
├── docker/       # Docker Compose stack + .env.example
└── docs/         # Network diagrams, notes
```

---

## 🐳 Services

| Service | Type | Host | Purpose |
|---------|------|------|---------|
| AdGuard Home | LXC | `pve-01` | DNS / ad blocking |
| Zoraxy | LXC | `pve-01` | Reverse proxy |
| Homepage | LXC | `pve-01` | Homelab dashboard |
| RustDesk Server | LXC | `pve-01` | Self-hosted remote desktop |
| Jellyfin | VM (GPU passthrough) | `pve-02` | Media server |
| qBittorrent | Docker | `pve-02` | Download client (ProtonVPN via WireGuard) |
| Radarr | Docker | `pve-02` | Movie management |
| Sonarr | Docker | `pve-02` | TV management |
| Prowlarr | Docker | `pve-02` | Indexer management |
| Profilarr | Docker | `pve-02` | Quality profile management |
| Mousehole | Docker | `pve-02` | Proxy sidecar for qBittorrent |
| Cross-seed | Docker | `pve-02` | Automated cross-seeding |
| Audiobookshelf | Docker | `pve-02` | Audiobook & podcast server |

> LXCs are provisioned using [Proxmox Helper Scripts](https://community-scripts.org/).

---

## 🐳 Docker Stack

All Docker services run inside a **single dedicated VM on `pve-02`**, defined in one `docker-compose.yml`. See [`docker/`](docker/) for the full Compose file and setup instructions.

### Container Images
some containers in my arr stack use [hotio](https://hotio.dev) images. check their site if you're configuring these for the first time.

### VPN
qBittorrent routes all traffic through **ProtonVPN via WireGuard** using hotio's built-in VPN support. You'll need a WireGuard config file from ProtonVPN — see [hotio's qBittorrent docs](https://hotio.dev/containers/qbittorrent/) for setup.

### Data Layout
The stack uses the [TRaSH Guides](https://trash-guides.info) hardlink folder structure so Radarr/Sonarr can move completed downloads without duplicating disk usage:
```
/mnt/data/
├── torrents/
│   ├── movies/
│   ├── tv/
│   └── books/
└── media/
    ├── movies/
    ├── books/
    └── tv/
```

---

*Built with ☕ and too many late nights.*
