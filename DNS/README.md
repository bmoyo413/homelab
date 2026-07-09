# 🕳️ Pi-hole

## What This Is

Two Pi-hole instances acting as **DNS servers** for my home network, one on `pve-01` and
one on `pve-02` for redundancy. This setup is purely for DNS and ad blocking; DHCP stays on
the router, not Pi-hole.

Alongside each Pi-hole I run **Unbound** as a recursive DNS resolver, so queries go straight
to the root DNS servers with no third-party upstream (Cloudflare, Google, etc.) ever seeing my
traffic. This is a personal preference, not a requirement (see the Unbound note below for the
tradeoff).

Both instances are kept in sync with **Nebula Sync**: any change to blocklists, allowlists, or
settings on the primary is replicated to the secondary automatically. See
[Nebula Sync](#-nebula-sync) at the end.

---

## 🖥️ Container Specs

| Setting | Value |
|---------|-------|
| CPU | 1 core |
| RAM | 512MB |
| Type | Unprivileged LXC |
| IP | Static (see the note under LXC Creation) |

---

## 📦 LXC Creation

Pi-hole is provisioned using [Proxmox Helper Scripts](https://community-scripts.github.io/ProxmoxVE/).

> 💡 Give the container a **static IP**. Through the **Advanced** install option you can set the
> IP, hostname, and other settings before the container is created.

Run this in the Proxmox shell:

```bash
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/ct/pihole.sh)"
```

> ⚠️ **A static IP on the container is not enough on its own.** Your router will still try to
> lease that address to other devices unless you either set it **outside the router's DHCP range**
> or add a **DHCP reservation** for it. Otherwise you can hit an IP clash that quietly breaks DNS
> for the whole network.

---

## 📋 Script Prompts

The script asks a few questions. Here is what to pick and why:

### 1. External Source Warning
```
This script sources external scripts. Are you sure you want to continue?
```
**→ Yes**, this is expected; the helper script pulls the official Pi-hole installer.

### 2. 🔵 Unbound (Optional)
```
Would you like to add Unbound?
```
**→ Yes** (my preference). Unbound runs as a recursive resolver alongside Pi-hole so no upstream
provider (Cloudflare, Google, Quad9) ever sees your queries.

> **Tradeoff:** recursive resolution is more private, but you become your own resolver, so
> first-time lookups can be a touch slower until the cache warms and there is a little more to
> maintain. If you would rather keep it simple, choose **No** and point Pi-hole at a public
> upstream instead (see the DNS settings below). Either way is fine.
>
> **If you chose No**, look for the 🔵 marker through the rest of this doc and skip those
> sections, they only apply if you installed Unbound.

### 3. 🔵 Unbound Mode
```
Would you like Unbound configured as a forwarding DNS server (DoT)?
```
**→ No**, keep it recursive. Recursive mode queries the root DNS servers directly, so no upstream
provider sees your queries. DoT (forwarding) still relies on a third party, just encrypted in
transit, which defeats the point of running Unbound.

---

## 🔑 Post-Install

The script does **not** set a Pi-hole web UI password. Run this inside the container after install:

```bash
pihole setpassword
```

---

## ⚙️ GUI Settings

Access the web UI at `http://<your-pihole-ip>/admin`.

> Some settings are hidden behind **Expert Mode**. Click **Settings** in the sidebar, then toggle
> **Expert Mode** in the top right before changing anything below.

### Settings → DNS

| Setting | Value |
|---------|-------|
| Upstream DNS | 🔵 `127.0.0.1#5335` (Unbound), uncheck everything else. If you did not install Unbound, pick one of the common upstreams below instead. |
| Enable DNSSEC | 🔵 ✅ On |
| Never forward non-FQDN A and AAAA queries | ✅ On |
| Never forward reverse lookups for private IP ranges | ✅ On |

> **Common upstream DNS options (non-Unbound only):**
> | Provider | Primary | Secondary |
> |----------|---------|-----------|
> | Cloudflare (privacy focused) | `1.1.1.1` | `1.0.0.1` |
> | Google | `8.8.8.8` | `8.8.4.4` |
> | Quad9 (blocks malicious domains) | `9.9.9.9` | `149.112.112.112` |

### Settings → DHCP

- Leave DHCP **off**: the router handles DHCP. Set Pi-hole as the DNS server in your router's DHCP
  settings instead.

### Settings → Privacy

- Set to **Show everything**, this is a personal homelab.

---

## 🚫 Blocklists

Go to **Group Management → Adlists** to add lists. After adding them, go to
**Tools → Update Gravity** to pull them in.

| List | Description | URL |
|------|-------------|-----|
| Steven Black Unified | A well established hosts-format list that consolidates several reputable sources into one. Covers ads, malware, and tracking. Good low false-positive general purpose list. | `https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts` |
| Hagezi Multi Normal | Actively maintained list with multiple tiers. Covers ads, tracking, telemetry, and some malware. Good balance of coverage and false positives at the normal tier. | `https://raw.githubusercontent.com/hagezi/dns-blocklists/main/adblock/multi.txt` |
| OISD Big | Large well-maintained list focused on ads and tracking. Known for being aggressive but with low false positives. | `https://big.oisd.nl` |

> **Hagezi tiers**, swap `multi.txt` for another level if needed:
> - `multi-light.txt`: light, very low false positives
> - `multi.txt`: normal, recommended starting point ✅
> - `multi-pro.txt`: more aggressive
> - `multi-pro++.txt`: aggressive, higher false-positive risk

---

## 🔁 Repeat for Second Instance

Repeat this whole process on the second Proxmox host for `pihole-02`. Each container runs its own
independent Pi-hole and Unbound instance; this is intentional for redundancy.

> Once both are up, use **Nebula Sync** to keep them in sync automatically instead of managing
> settings on each one separately. See [Nebula Sync](#-nebula-sync) below; after that, make all
> config changes on the primary instance only.

---

## 🔄 Nebula Sync

Nebula Sync keeps both Pi-hole instances in sync automatically. Any change made on `pihole-01`
(blocklists, allowlists, settings) is replicated to `pihole-02`.

- Repo: [lovelaze/nebula-sync](https://github.com/lovelaze/nebula-sync)

Nebula Sync runs as a container in the media stack (`docker/media/docker-compose.yml`), driven by
`PIHOLE_PRIMARY_IP` / `PIHOLE_REPLICA_IP` and the two web passwords in that stack's `.env`. The
replica's web password must equal `NEBULA_REPLICA_PASS`, or sync fails to authenticate (the logs
show a connection or 401 error against the replica).

> ⚠️ **What Nebula Sync does NOT carry:** it replicates the Teleporter export (lists, settings,
> `pihole.toml`) but **skips files under `/etc/dnsmasq.d/`**. The local-DNS wildcard lives there
> (see below), so it does **not** sync. Apply that record on **both** instances by hand.

---

## 🌐 Local DNS (split-horizon)

To give my internal services friendly names, each Pi-hole carries a one-line dnsmasq wildcard so
`*.<your-domain>` resolves to the internal Caddy / apps-VM IP on the LAN instead of leaking out to
public DNS. (This pattern is called split-horizon DNS: the same name resolves differently inside
the network than it would outside.) The committed template is
[`dnsmasq.d/local-dns.conf.example`](dnsmasq.d/local-dns.conf.example):

```
address=/<your-domain>/<apps-VM-IP>
```

Install it on **each** Pi-hole (Nebula Sync does not replicate it):

```bash
# inside the Pi-hole LXC
echo 'address=/<your-domain>/<apps-VM-IP>' > /etc/dnsmasq.d/local-dns.conf
service pihole-FTL restart        # NOT `pihole reloaddns`: a SIGHUP does not re-read /etc/dnsmasq.d/
```

Notes:
- Pi-hole v6 only loads `/etc/dnsmasq.d/*.conf` when `misc.etc_dnsmasq_d` is `true`. Check with
  `pihole-FTL --config misc.etc_dnsmasq_d`; enable with `pihole-FTL --config misc.etc_dnsmasq_d true`
  if needed.
- This build has **no** `pihole restartdns`. Use `service pihole-FTL restart` (or `pihole reloaddns`
  only when changing lists, not config files).
- Validate the file with `pihole-FTL dnsmasq-test` before restarting.

---

## ♻️ Rebuilding a lost instance

If a host is reinstalled and its Pi-hole LXC is lost (this happened to my secondary during a
storage rebuild), the primary stays the source of truth, so recovery is quick:

1. Recreate the LXC with the same **static IP** ([LXC Creation](#-lxc-creation), Unbound: Yes /
   recursive).
2. Set its web password to the existing `NEBULA_REPLICA_PASS` from the media stack `.env`
   (`pihole setpassword`).
3. Let Nebula Sync run (or force it: `docker restart nebula-sync`); it repopulates blocklists,
   allowlists, and settings from the primary.
4. Reinstall the local-DNS wildcard ([above](#-local-dns-split-horizon)): Nebula Sync does not
   carry it.
5. Verify: `nslookup <a-subdomain>.<your-domain> <secondary-IP>` resolves to the apps-VM IP, and
   the router still hands out **both** Pi-hole IPs for DNS (redundancy + failover).
