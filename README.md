# 🏠 Homelab (WIP)

![GitHub last commit](https://img.shields.io/github/last-commit/bmoyo413/homelab)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

Personal homelab running on two Proxmox hosts. Services run across Docker containers, LXCs (provisioned via Proxmox Helper Scripts), and dedicated VMs.

---

## 🖥️ Hardware

| Host | CPU | GPU | RAM | Storage |
|------|-----|-----|-----|---------|
| `pve-01` | Ryzen 5 2600X | RX 580 | 32GB DDR4 | 4TB Mirror (2x 4TB) |
| `pve-02` | Ryzen 5 2600 | GTX 1660 (passthrough) | 24GB  DDR4 | ZFS RAIDZ2 (6x 1TB) + 3x 256GB SSD Stripe |

---

## 🗂️ Repo Structure
```
homelab/
├── DNS/          # Dns config
├── ansible       # Automation
└── docker        # Docker Compose stack 
```

---

## 🐳 Services

| Service | Type | Host | Purpose |
|---------|------|------|---------|
| Pi-Hole + Ubound| LXC | `pve-01` | Primary DNS / ad blocking |
| Pi-Hole + Ubound| LXC | `pve-02` | Seconday DNS / ad blocking |
| Zoraxy | LXC | `pve-01` | Reverse proxy |
| Homepage | LXC | `pve-01` | Homelab dashboard |
| RustDesk Server | LXC | `pve-01` | Self-hosted remote desktop |
| Ansible / Semaphore | LXC | `pve-01` | Automation & playbook management UI |
| Jellyfin | VM (GPU passthrough) | `pve-02` | Media server |
| qBittorrent | Docker | `pve-02` | Download client (ProtonVPN via WireGuard) |
| Radarr | Docker | `pve-02` | Movie management |
| Sonarr | Docker | `pve-02` | TV management |
| Prowlarr | Docker | `pve-02` | Indexer management |
| Profilarr | Docker | `pve-02` | Quality profile management |
| Mousehole | Docker | `pve-02` | Proxy sidecar for qBittorrent |
| Cross-seed | Docker | `pve-02` | Automated cross-seeding |
| Audiobookshelf | Docker | `pve-02` | Audiobook & podcast server |
| Nebula-Sync | Docker | `pve-02` | Pi-hole config sync between instances |
| Prometheus | LXC | `pve-02` | Metrics collection |
| Grafana | LXC | `pve-02` | Metrics dashboards & visualization |
| Loki | LXC | `pve-02` | Log aggregation |
| Gitea | LXC | `pve-02` | Self-hosted Git server |
| Vaultwarden | LXC | `pve-02` | Self-hosted password manager (Bitwarden compatible) |


> LXCs are provisioned using [Proxmox Helper Scripts](https://community-scripts.org/).

---

*Built with ☕ and too many late nights.*
