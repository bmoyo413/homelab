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

