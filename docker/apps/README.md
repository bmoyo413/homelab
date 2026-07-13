# Apps stack

A declarative Compose stack for a dedicated Docker VM on **`pantheon` (pve-02)**,
with appdata on the **`dagger`** pool. Same conventions as the
[media stack](../media/): every image pinned, `no-new-privileges`, restart
policies, healthchecks where the image supports them, and secrets in a gitignored
`.env`. Built incrementally; the monitoring core **and** the core apps are live.

## Live

| Service | Image | Purpose |
|---------|-------|---------|
| Caddy | `homelab/caddy` (custom build) | Reverse proxy + **real Let's Encrypt certs** (Cloudflare DNS-01); only service that publishes ports |
| Prometheus | `prom/prometheus` | Metrics collection (30d retention) + alert rules |
| Alertmanager | `prom/alertmanager` | Routes/dedupes alerts → ntfy bridge |
| ntfy-alertmanager | `xenrox/ntfy-alertmanager` | Bridge: formats Alertmanager alerts into ntfy messages |
| Grafana | `grafana/grafana` | Dashboards (as code) + Prometheus/Loki datasources auto-provisioned |
| Loki | `grafana/loki` | Log aggregation (single-binary, filesystem) |
| Grafana Alloy | `grafana/alloy` | Tails all Docker container logs → Loki (read-only socket) |
| node-exporter | `prom/node-exporter` | Apps-VM OS metrics |
| cAdvisor | `gcr.io/cadvisor/cadvisor` | Per-container metrics for the apps host (internal scrape) |
| pve-exporter | `prompve/prometheus-pve-exporter` | Proxmox cluster metrics (both nodes, via the PVE API) |
| blackbox-exporter | `prom/blackbox-exporter` | Probes each routed HTTPS endpoint → TLS cert expiry + reachability |
| ntfy | `binwiederhier/ntfy` | Push notifications + the alert notifier |
| Gitea | `gitea/gitea` | Self-hosted Git (SQLite; HTTPS cloning via Caddy) |
| Vaultwarden | `vaultwarden/server` | Password manager (Bitwarden-compatible) |
| Authentik | `ghcr.io/goauthentik/server` | **SSO / OIDC provider** (server + worker + Postgres + Redis); Grafana and Gitea sign in through it |
| CouchDB | `couchdb` | Sync backend for **Obsidian** Self-hosted LiveSync (real-time, E2E-encrypted notes) |
| Obsidian webtop | `lscr.io/linuxserver/obsidian` | Obsidian desktop in the browser (KasmVNC); a second LiveSync client of CouchDB |
| Homepage | `ghcr.io/gethomepage/homepage` | **Dashboard**: single pane of glass over every service; config-as-code (`./homepage/*.yaml`) |
| socket-proxy | `ghcr.io/tecnativa/docker-socket-proxy` | Read-only Docker API gateway for Homepage (only CONTAINERS + INFO; no raw socket mount) |

**Scrape targets** (layered, one job each):
- **node-exporter** (OS per VM): apps VM (internal) + media VM (cross-host `:9100`)
- **cAdvisor** (containers): apps host (internal) + media host (cross-host `:8081`)
- **pve** (hypervisor): both Proxmox cluster nodes via the PVE API (one token)
- **blackbox-tls** (**TLS cert expiry**): each routed HTTPS endpoint, probed through Caddy

Apps-side exporters are scraped internally by name; the cross-host targets (media VM,
the two PVE nodes) and the blackbox FQDNs are sanitized placeholders in
[`prometheus/prometheus.yml`](prometheus/prometheus.yml); edit them to your LAN IPs /
domain (Prometheus can't read `.env`).

## Alerting (observability as code)

`prometheus/rules/*.yml` → **Prometheus** evaluates → **Alertmanager** groups/dedupes
→ **ntfy-alertmanager** formats → **ntfy** (topic `homelab-alerts`). Rules
([`prometheus/rules/alerts.yml`](prometheus/rules/alerts.yml)): target down, disk >85%,
memory >90%, CPU >90%, config-reload failure, and **TLS cert expiry (<14d / <3d) + probe
failure**. Subscribe to the `homelab-alerts` topic in the ntfy app.

## Dashboards as code

Grafana auto-loads JSON from [`grafana/dashboards/`](grafana/dashboards/) (mounted
read-only). Ships with **Node Exporter Full**; datasources are pinned by UID
(`prometheus`/`loki`) so provisioned dashboards bind deterministically. Add more by
dropping JSON in that folder.

## Exposure & TLS

**LAN-only, no public ports.** Only Caddy maps `80`/`443`; everything else is
internal-only and reached through Caddy by container name. Caddy obtains **real
Let's Encrypt certs via the Cloudflare DNS-01 challenge** (no inbound ports), so there
are **no cert warnings**. Pair with **split-horizon DNS**: point `*.${APPS_DOMAIN}` at
this host in Pi-hole so traffic stays on the LAN. Routed subdomains:
`grafana.`, `prometheus.`, `alertmanager.`, `ntfy.`, `gitea.`, `vault.`, `auth.`, `obsidian.`,
`obsidian-app.`, `homepage.${APPS_DOMAIN}`.

## Single sign-on (Authentik)

Authentik is the central **OIDC provider**; **Grafana and Gitea authenticate through it**.
The OAuth2 providers and applications are declared as **blueprints**
([`authentik/blueprints/`](authentik/blueprints/)) and applied on boot, so the SSO config is
reproducible rather than clicked into a UI. Local admin logins stay as a break-glass fallback.

## Obsidian sync (CouchDB)

CouchDB is the backend for the **Obsidian Self-hosted LiveSync** plugin: real-time,
end-to-end-encrypted note sync across desktop and mobile, served at
`obsidian.${APPS_DOMAIN}` through Caddy. It uses CouchDB's own basic auth (not Authentik).
On the LAN an iPhone syncs directly against CouchDB; off-network sync is a planned Tailscale
overlay (see the roadmap in the top-level README), not live yet.
A browser-accessible **Obsidian webtop** (`obsidian-app`, KasmVNC) runs the desktop app at
`obsidian-app.${APPS_DOMAIN}` as a second LiveSync client of the same CouchDB.

## Dashboard (Homepage)

[Homepage](https://gethomepage.dev) is the single pane of glass over the lab, served at
`homepage.${APPS_DOMAIN}`. Its whole config is committed flat YAML in
[`homepage/`](homepage/) (services, settings, widgets, bookmarks); host-specific URLs and any
optional widget tokens come from `.env` as `HOMEPAGE_VAR_*`, so the committed config stays
host-agnostic and secret-free. Live container status comes from a read-only
`docker-socket-proxy` sidecar (only `CONTAINERS` + `INFO` exposed) rather than mounting the raw
Docker socket into the dashboard.

## Deploy

One-time setup on the VM (Docker VM on `pantheon`, `dagger` mounted at `${APPDATA}`):

1. Create + chown the per-service appdata dirs (grafana `472`, loki `10001`, gitea `${PUID}`;
   vaultwarden runs as root).
2. Point Pi-hole `*.${APPS_DOMAIN}` at this VM (split-horizon DNS).
3. Fill in `.env`: `CF_API_TOKEN` (Cloudflare Zone → DNS → Edit) + `ACME_EMAIL`, a read-only
   Proxmox API token for pve-exporter, and a Vaultwarden `ADMIN_TOKEN`
   (`docker run --rm vaultwarden/server /vaultwarden hash`).
4. Set the real IPs in `prometheus/prometheus.yml` (Prometheus can't read `.env`).

Then, from the repo root:

```bash
cp .env.example .env                                          # fill in secrets
docker compose -f docker/apps/docker-compose.yml config       # parse-check
docker compose -f docker/apps/docker-compose.yml up -d        # start
```

On first start, create your Vaultwarden and Gitea accounts (open registration is disabled),
then leave signups off.

## Planned (later)
- **Services:** immich, stirling-pdf, memos (if wanted).
