# 🏠 Homelab

![GitHub last commit](https://img.shields.io/github/last-commit/bmoyo413/homelab)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

My self-hosted homelab: two Proxmox VE hosts, ZFS storage, and a declarative Docker
stack that all lives in git. The idea is **reproducible infrastructure**, everything
is described as code (Compose, CI, more to come) instead of hand-built and forgotten.

> **Status:** the media and apps stacks are up and running, apps behind real Let's
> Encrypt certs (Cloudflare DNS-01) with Prometheus/Grafana/Loki monitoring across
> both nodes and Authentik single sign-on. Still very much a work in progress, more
> pieces get added and documented as I go.

---

## 🖥️ Hosts

Two Proxmox VE nodes, named **`joker`** (`pve-01`) and **`pantheon`** (`pve-02`).

| Host | Node | CPU | GPU | RAM | ZFS storage |
|------|------|-----|-----|-----|-------------|
| `joker` | `pve-01` | Ryzen 5 2600X | RX 580 | 32 GB DDR4 | `tank`: 2×4 TB mirror (media) |
| `pantheon` | `pve-02` | Ryzen 5 2600 | GTX 1660 (passthrough) | 24 GB DDR4 | `dagger`: SSD raidz1 (VM/app data) · `shield`: HDD raidz1 (bulk storage) |

---

## 🗺️ What runs where (current)

### `joker` (pve-01)
| Workload | Type | Purpose |
|----------|------|---------|
| **Docker media stack** (`medsev` VM) | VM | The full arr/download stack, see [below](#-docker-media-stack) |
| Pi-hole + Unbound | LXC | Primary DNS / ad-blocking (recursive Unbound) |
| RustDesk Server | LXC | Self-hosted remote desktop relay |

### `pantheon` (pve-02)
| Workload | Type | Purpose |
|----------|------|---------|
| **Docker apps stack** (apps VM) | VM | Caddy, monitoring, Gitea, Vaultwarden, ntfy, Authentik (SSO), CouchDB (Obsidian sync), Homepage dashboard: **live** (appdata on `dagger`) |
| Jellyfin | VM (GPU passthrough) | Media server |
| Pi-hole + Unbound | LXC | Secondary DNS (config replicated from primary via Nebula-Sync) |

> LXCs are provisioned with [Proxmox Helper Scripts](https://community-scripts.org/).
> DHCP stays on the router; the two Pi-holes provide redundant DNS.

---

## 🐳 Docker media stack

A single declarative Compose stack ([`docker/media/docker-compose.yml`](docker/media/docker-compose.yml))
running in the `medsev` VM on `joker`. Images are pinned to explicit versions, except the
hotio arr images, which track `:release` because upstream prunes old versioned tags. All
services run with `no-new-privileges` and carry restart policies; the download/arr core has
healthchecks and startup ordering.

| Service | Image | Purpose |
|---------|-------|---------|
| qBittorrent | hotio | Download client: all traffic via **ProtonVPN/WireGuard** |
| Radarr / Sonarr / Prowlarr | hotio | Movie / TV / indexer management |
| Profilarr | santiagosayshey | Quality-profile management (TRaSH) |
| Flaresolverr | flaresolverr | Cloudflare-challenge solver for Prowlarr |
| Mousehole | tmmrtn | qBittorrent companion (shares its network namespace) |
| Cross-seed | cross-seed | Automated cross-seeding |
| Audiobookshelf | advplyr | Audiobook & podcast server |
| Nebula-Sync | lovelaze | Replicates Pi-hole config primary to secondary |
| cAdvisor | google | Per-container metrics (for the apps-stack Prometheus) |
| node-exporter | prom | This VM's OS metrics (for the apps-stack Prometheus) |

Setup, the VPN kill-switch, and the TRaSH hardlink layout are documented in
[`docker/media/README.md`](docker/media/README.md). Secrets live in a gitignored `docker/media/.env`
(template: `docker/media/.env.example`); nothing sensitive is committed.

---

## 🐳 Docker apps stack

A single declarative Compose stack ([`docker/apps/docker-compose.yml`](docker/apps/docker-compose.yml))
running in a dedicated apps VM on `pantheon`, appdata on the `dagger` SSD pool. Same conventions
as the media stack: image-pinned, `no-new-privileges`, healthchecks. Only Caddy publishes ports;
every other service is internal-only and reached by container name.

| Service | Image | Purpose |
|---------|-------|---------|
| Caddy | custom (Caddy + Cloudflare DNS plugin) | Reverse proxy; real Let's Encrypt certs via DNS-01 |
| Authentik | goauthentik | Single sign-on (OIDC) for Grafana, Gitea, and the Proxmox hosts |
| Prometheus / Alertmanager / Grafana | prom / grafana | Metrics, dashboards, and alerting routed to ntfy |
| Loki / Alloy | grafana | Log aggregation and shipping |
| node-exporter / cAdvisor / pve-exporter / blackbox-exporter | prom | Host, container, Proxmox, and TLS/cert probes |
| Gitea | gitea | Self-hosted git, SQLite backend |
| Vaultwarden | dani-garcia | Password manager, Bitwarden-compatible |
| CouchDB | apache | Obsidian LiveSync backend |
| Obsidian webtop | linuxserver | Browser-accessible Obsidian desktop (KasmVNC) |
| Homepage | gethomepage | Dashboard for the whole lab, config-as-code |
| ntfy | binwiederhier | Push notifications for alerts |

Setup, SSO, and monitoring are documented in
[`docker/apps/README.md`](docker/apps/README.md). Secrets live in a gitignored
`docker/apps/.env` (template: `docker/apps/.env.example`); nothing sensitive is committed.

---

## 🗂️ Repo structure
```
homelab/
├── DNS/            # Pi-hole + Unbound build guide
├── docker/
│   ├── media/      # Declarative media stack (arr + qBittorrent/VPN)
│   └── apps/       # Apps stack (Caddy, monitoring, Gitea, Vaultwarden, ntfy, Authentik, CouchDB)
└── .github/        # CI: validates Compose stacks
```

---

## 🔧 On the workbench

A homelab is never really "done," and this repo is honest about that. Things I'm
actively working toward, roughly in order:

- [ ] **Ansible into the repo:** publish the fleet-patching playbook + inventory so
      routine updates across the nodes are as-code, not by hand.
- [ ] **Provisioning as-code (Terraform):** define the Proxmox VMs and LXCs declaratively
      instead of clicking through helper scripts, so a node can be rebuilt from config.
- [ ] **Backups:** planned, off-host to start, off-site down the line.
- [ ] **Tailscale mesh, as-code:** manage remote access to the lab as a defined overlay
      network instead of hand-rolled port-forwards.
- [ ] **Runbooks:** per-service setup + recovery notes, added as each piece stabilizes.

If something here looks half-finished, it probably is. That's the point: this is a
living lab I keep tightening, not a snapshot I polished once and walked away from.

---

*Built with ☕ and too many late nights.*
