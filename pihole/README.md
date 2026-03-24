# 🕳️ Pi-hole

## What This Is

Two Pi-hole instances running as **DNS-only servers** for the home network. One on `pve-01`, one on `pve-02` for redundancy. This setup is purely for DNS and ad blocking DHCP is handled by the router, not Pi-hole.

Unbound runs alongside each Pi-hole instance as a recursive DNS resolver, meaning queries go directly to root DNS servers with no third-party upstream (Cloudflare, Google, etc.) ever seeing your traffic.

Both instances are kept in sync using **Nebula Sync** any changes to blocklists, allowlists, or settings on the primary instance are automatically replicated to the secondary. See [Nebula Sync](#-nebula-sync) at the end of this doc.

---

## 🖥️ Container Specs

| Setting | Value |
|---------|-------|
| CPU | 1 core |
| RAM | 512MB |
| Type | Unprivileged LXC |
| IP | Static |

---

## 📦 LXC Creation

Pi-hole is provisioned using [Proxmox Helper Scripts](https://community-scripts.github.io/ProxmoxVE/).

> 💡 It is recommended to check that Pi-hole is assigned a **static IP** after creation. If you go through the **Advanced** install option during LXC creation you can configure the IP, hostname, and other settings before the container is created.

Run the following in the Proxmox shell:

```bash
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/ct/pihole.sh)"
```

---

## 📋 Script Prompts

The script will ask several questions. Here is what to choose and why:

### 1. External Source Warning
```
This script sources external scripts. Are you sure you want to continue?
```
**→ Yes** this is expected, the helper script pulls the official Pi-hole installer.

### 2. 🔵 Unbound (Optional)
```
Would you like to add Unbound?
```
**→ Yes** (recommended) installs Unbound as a recursive DNS resolver alongside Pi-hole.

> **If you chose No** look for the 🔵 marker throughout this doc and skip those sections, they only apply if you installed Unbound.

### 3. 🔵 Unbound Mode
```
Would you like Unbound configured as a forwarding DNS server (DoT)?
```
**→ No** keep it as recursive. Recursive mode queries root DNS servers directly so no upstream provider ever sees your queries. Choosing DoT (forwarding) still relies on a third party, just encrypted in transit which defeats the point of running Unbound.

---

## 🔑 Post-Install

The script does **not** set a Pi-hole web UI password automatically. Run this inside the container after install:

```bash
pihole setpassword
```

---

## ⚙️ GUI Settings

Access the web UI at `http://<your-pihole-ip>/admin`.

> Some settings are hidden behind **Expert Mode** by default. Click **Settings** in the sidebar, then toggle **Expert Mode** in the top right before making any changes below.

### Settings → DNS

| Setting | Value |
|---------|-------|
| Upstream DNS | 🔵 `127.0.0.1#5335` (Unbound) Uncheck everything else. If you did not install Unbound pick one of the common upstreams below instead. |
| Enable DNSSEC | 🔵 ✅ On |
| Never forward non-FQDN A and AAAA queries | ✅ On |
| Never forward reverse lookups for private IP ranges | ✅ On |

> **Common upstream DNS options (non-Unbound only):**
> | Provider | Primary | Secondary |
> |----------|---------|-----------|
> | Cloudflare (recommended, privacy focused) | `1.1.1.1` | `1.0.0.1` |
> | Google | `8.8.8.8` | `8.8.4.4` |
> | Quad9 (blocks malicious domains) | `9.9.9.9` | `149.112.112.112` |

### Settings → DHCP

- Leave DHCP **off** the router handles DHCP. Set Pi-hole as the DNS server in your router's DHCP settings instead.

### Settings → Privacy

- Set to **Show everything** this is a personal homelab.

---

## 🚫 Blocklists

Go to **Group Management → Adlists** in the sidebar to add lists. After adding all lists, go to **Tools → Update Gravity** to pull them in.

| List | Description | URL |
|------|-------------|-----|
| Steven Black Unified | A well established hosts-format list that consolidates several reputable sources into one. Covers ads, malware, and tracking. Good low false-positive general purpose list. | `https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts` |
| Hagezi Multi Normal | Actively maintained list with multiple tiers. Covers ads, tracking, telemetry, and some malware. Good balance of coverage and false positives at the normal tier. | `https://raw.githubusercontent.com/hagezi/dns-blocklists/main/adblock/multi.txt` |
| OISD Big | Large well-maintained list focused on ads and tracking. Known for being aggressive but with low false positives. | `https://big.oisd.nl` |

> **Hagezi tiers** >> swap `multi.txt` for a different level if needed:
> - `multi-light.txt` >> light, very low false positives
> - `multi.txt` >> normal, recommended starting point ✅
> - `multi-pro.txt` >> more aggressive
> - `multi-pro++.txt` >> aggressive, higher false positive risk

---

## 🔁 Repeat for Second Instance

Repeat this entire process on the second Proxmox host for `pihole-02`. Each container runs its own independent Pi-hole and Unbound instance this is intentional for redundancy.

> Once both instances are up, use **Nebula Sync** to keep them in sync automatically rather than managing settings on each instance separately. See [Nebula Sync](#-nebula-sync) below all config changes should be made on the primary instance only after that is set up.

---

## 🔄 Nebula Sync

Nebula Sync keeps both Pi-hole instances in sync automatically. Any changes made on `pihole-01` (blocklists, allowlists, settings) are replicated to `pihole-02`.

- Repo: [lovelaze/nebula-sync](https://github.com/lovelaze/nebula-sync)
