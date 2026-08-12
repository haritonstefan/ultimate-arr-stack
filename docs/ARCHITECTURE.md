# Architecture Overview

This guide explains how the stack fits together and why it's designed this way.

## The Request-to-Watch Flow

When someone requests a movie or TV show, here's what happens:

```
┌─────────────┐     ┌──────────────┐     ┌───────────┐     ┌─────────────┐     ┌──────────┐
│   Seerr     │────▶│ Sonarr/Radarr│────▶│ Prowlarr  │────▶│ qBittorrent │────▶│ Jellyfin │
│ (request)   │     │ (manage)     │     │ (indexers)│     │   SABnzbd   │     │ (watch)  │
│             │     │              │     │           │     │ (download)  │     │          │
└─────────────┘     └──────────────┘     └───────────┘     └─────────────┘     └──────────┘
```

1. **Seerr** - User requests a show or movie
2. **Sonarr/Radarr** - Searches for releases, sends to download client
3. **Prowlarr** - Provides indexers (torrent + Usenet) to Sonarr/Radarr
4. **qBittorrent** - Downloads torrents
5. **SABnzbd** - Downloads from Usenet
6. **Jellyfin** - Streams the completed files

> **Why both qBittorrent and SABnzbd?** Torrents are free but can be slow/unreliable. Usenet costs ~$5/month but is faster, more reliable, and has no ratio requirements. Most users configure both - Sonarr/Radarr will try Usenet first, fall back to torrents.

## Service Connections

Every service runs on the `arr-stack` bridge network with its own static IP, so they all reach each other by container name — no shared network namespace, no `localhost` tricks.

```
Bridge → bridge (use name):
─────────────────────────────
Sonarr → qBittorrent            Prowlarr → Sonarr
  └── qbittorrent:8085             └── sonarr:8989
Radarr → SABnzbd                 Prowlarr → Radarr
  └── sabnzbd:8080                  └── radarr:7878
Seerr/Bazarr → Sonarr            Prowlarr → FlareSolverr
  └── sonarr:8989 / radarr:7878     └── flaresolverr:8191
```

## Network Layout

All services run on the `arr-stack` network with static IPs:

```
arr-stack network (172.20.0.0/24)
───────────────────────────────────────────────────────────────────────────────────
│ IP           │ Service      │ Notes                          │ Required for     │
├──────────────┼──────────────┼────────────────────────────────┼──────────────────│
│ 172.20.0.4   │ Jellyfin     │ Media server                   │ Core             │
│ 172.20.0.8   │ Seerr        │ Request portal                 │ Core             │
│ 172.20.0.9   │ Bazarr       │ Subtitles                      │ Core             │
│ 172.20.0.10  │ Sonarr       │ TV manager                     │ Core             │
│ 172.20.0.11  │ Radarr       │ Movie manager                  │ Core             │
│ 172.20.0.17  │ qBittorrent  │ Torrent client                 │ Core             │
│ 172.20.0.18  │ SABnzbd      │ Usenet client                  │ Core             │
│ 172.20.0.19  │ Prowlarr     │ Indexer manager                │ Core             │
│ 172.20.0.21  │ FlareSolverr │ Cloudflare bypass for Prowlarr  │ Core             │
│ 172.20.0.5   │ Pi-hole      │ DNS server                     │ Core             │
│ 172.20.0.2   │ Traefik      │ Reverse proxy                  │ + local DNS      │
│ 172.20.0.12  │ Cloudflared  │ Tunnel to Cloudflare           │ + remote access (Cloudflared) │
│ host-network │ Tailscale    │ Mesh VPN subnet router         │ + remote access (Tailscale)   │
│ 172.20.0.13  │ Uptime Kuma  │ Monitoring                     │ Optional         │
│ 172.20.0.14  │ duc          │ Disk usage                     │ Optional         │
│ 172.20.0.15  │ Beszel       │ System monitoring              │ Optional         │
│ 172.20.0.16  │ DIUN         │ Image update notifier          │ Optional         │
───────────────────────────────────────────────────────────────────────────────────
```

> `172.20.0.20` is reserved for Baserow (a separate, unrelated stack sharing this network) — never assign it here. `172.20.0.128/25` is confined to dynamically-assigned IPs, keeping them away from every statically-pinned service above.

## Access Levels

```
┌─────────────────────────────────────────────────────────────────────────┐
│                             CORE                                         │
│                      Access via NAS_IP:port                              │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐            │
│  │ :8096     │  │ :8989     │  │ :7878     │  │ :5055     │  ...       │
│  │ Jellyfin  │  │ Sonarr    │  │ Radarr    │  │   Seerr   │            │
│  └───────────┘  └───────────┘  └───────────┘  └───────────┘            │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ + Pi-hole + Traefik
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                          + LOCAL DNS                                     │
│                Access via .lan domains (+ .local, client-dependent)      │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐               │
│  │ jellyfin.lan  │  │ sonarr.lan    │  │ radarr.lan    │  ...          │
│  └───────────────┘  └───────────────┘  └───────────────┘               │
│                                                                          │
│  Your device → Pi-hole (DNS) → Traefik → Service                        │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ + Cloudflared and/or Tailscale
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        + REMOTE ACCESS                                   │
│                   Access from outside your home                          │
│                                                                          │
│  ┌─── Path a: Cloudflared (public HTTPS via your domain) ──────────┐    │
│  │                                                                  │    │
│  │  ┌─────────────────────┐  ┌─────────────────────┐               │    │
│  │  │ jellyfin.domain.com │  │ seerr.domain.com     │  ...          │    │
│  │  └─────────────────────┘  └─────────────────────┘               │    │
│  │  Phone → Cloudflare → Tunnel → Traefik → Service                │    │
│  └──────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ┌─── Path b: Tailscale (private mesh VPN, full LAN access) ───────┐    │
│  │                                                                  │    │
│  │  ┌──────────────┐  ┌────────────────┐  ┌──────────────────────┐ │    │
│  │  │ sonarr.lan   │  │ pihole.lan     │  │ homeassistant.lan    │ │    │
│  │  └──────────────┘  └────────────────┘  └──────────────────────┘ │    │
│  │  Phone → Tailscale → LAN (10.10.0.0/24) → Service               │    │
│  │  (No public exposure; only authorised tailnet devices reach LAN)│    │
│  └──────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  Combinable: run either path or both.                                    │
└─────────────────────────────────────────────────────────────────────────┘
```

## Container Security

All containers run with hardened defaults:

- **`no-new-privileges`** — Prevents processes from gaining additional privileges via `setuid`/`setgid` binaries
- **`cap_drop: ALL`** — Drops all Linux capabilities by default

Two YAML anchors define security profiles in each compose file:

| Anchor | Used by | Capabilities |
|--------|---------|-------------|
| `x-security` | All non-LSIO services | None by default (services add back only what they need) |
| `x-security-lsio` | Sonarr, Radarr, Prowlarr, qBittorrent, SABnzbd, Bazarr | `CHOWN`, `SETUID`, `SETGID`, `DAC_OVERRIDE` (s6-overlay needs these to switch users during init) |

Services that write to Docker volumes as root add back `CHOWN` + `DAC_OVERRIDE` (Jellyfin, Seerr, Uptime Kuma, DUC, Beszel, DIUN, Configarr). Services with read-only or no volumes don't need any (FlareSolverr, Cloudflared, Traefik, Beszel-agent).

Additional requirements:
- **Uptime Kuma** — adds `FOWNER` (sets ownership on created files)
- **Pi-hole** — adds `NET_ADMIN`, `NET_RAW`, `CHOWN`, `SETUID`, `SETGID`, `SETFCAP`, `SYS_NICE`, `DAC_OVERRIDE`, and disables `no-new-privileges` (FTL uses `setcap` at startup)

## Design Decisions

**Static IPs:** Prevents "container not found" errors after restarts. Services always know where to find each other.

**Separate compose files:** Deploy only what you need. Core users don't need Traefik, Cloudflared, or Tailscale.

**Pi-hole for DNS:** Provides internal Docker DNS and ad-blocking. Optionally enables `.lan`/`.local` domains (+ local DNS).

**Named volumes:** Data persists across container updates. Easy to backup with the included script.

**No fail2ban:** External access goes through Cloudflare Tunnel, which handles rate limiting and bot protection at the edge. LAN services aren't exposed to the internet. Services with auth (qBittorrent, Pi-hole, Traefik dashboard) have their own brute-force protections. fail2ban would add complexity with no practical benefit.
