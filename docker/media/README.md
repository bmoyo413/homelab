# Media stack

The arr/download stack, in a single dedicated VM (**`medsev`**) on **`joker` (pve-01)**,
defined in one [`docker-compose.yml`](docker-compose.yml). Same conventions as the other
stacks: images pinned to explicit versions (the hotio arr images track `:release`, which
upstream prunes old versioned tags), `no-new-privileges`, restart policies, healthchecks on
the core, and secrets in a gitignored `.env`.

## Services

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
| cAdvisor | google | Per-container metrics, scraped by the apps-stack Prometheus (`:8081`) |
| node-exporter | prom | This VM's OS metrics, scraped by the apps-stack Prometheus (`:9100`) |

## Config & secrets

Secrets and host-specific values (timezone, LAN, Pi-hole IPs/passwords) stay out of git
in `.env` (gitignored; template `.env.example`). The qBittorrent WireGuard config
(`wg0.conf`) lives in its `/config` appdata volume and is **never** committed.

Heads up: Mousehole (v0.4.0+) only answers on localhost by default. Add whatever
hostname/domain or IP you actually browse its UI at to `MOUSEHOLE_ALLOWED_HOSTS`,
otherwise the page just won't load.

Some arr containers use [hotio](https://hotio.dev) images; check their site when
configuring for the first time.

## Deploy

```bash
cp .env.example .env                # then edit .env with your real values
docker compose config               # parse-check
docker compose up -d                # start in the background
docker compose logs -f qbittorrent  # follow one service
```

(Run these from inside `docker/media/`. Compose auto-loads `docker-compose.yml` and the
`.env` in the current directory.)

## VPN kill-switch

qBittorrent routes all traffic through **ProtonVPN via WireGuard** using hotio's built-in
VPN support: you supply a WireGuard config from ProtonVPN (see
[hotio's docs](https://hotio.dev/containers/qbittorrent/)). Companion containers
(Mousehole) share qBittorrent's network namespace, so they egress through the tunnel too.

## Data layout

The stack uses the [TRaSH Guides](https://trash-guides.info) hardlink folder structure so
Radarr/Sonarr move completed downloads without duplicating disk usage:

```
/mnt/data/
├── torrents/{movies,tv,books}
└── media/{movies,tv,books}
```
