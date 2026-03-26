# 🖥️ Self-Hosted Home NAS Server

> A privacy-first personal cloud server built on commodity desktop hardware — replacing commercial cloud storage with a fully owned, locally controlled alternative.

**Project by — Shambo Ghosh, Electronics and Communication Engineering Department**

---

## What This Is

A complete self-hosted Network Attached Storage (NAS) server running on a repurposed desktop PC. The goal was to eliminate subscription-based cloud storage (Google Drive, iCloud, etc.) and replace it with a private, locally controlled system that the whole family can use — with zero recurring cost and full data ownership.

---

## Tech Stack

| Layer | Technology |
|---|---|
| **OS** | Ubuntu Server 22.04 LTS |
| **Dashboard** | CasaOS |
| **Cloud Platform** | Nextcloud 33.0.0 (Docker) |
| **Database** | PostgreSQL 14.2 + Redis 6.2.20 |
| **Remote Access** | Tailscale VPN (WireGuard) |
| **HDD Monitoring** | smartmontools (S.M.A.R.T.) |
| **Container Runtime** | Docker (via CasaOS) |

---

## Architecture Overview

The system follows a layered design:

- **CasaOS** handles container orchestration and provides a simple dashboard UI
- **Nextcloud** (Docker) is the application layer — file storage, photo backup, multi-user access
- **Tailscale** creates a zero-config WireGuard VPN mesh — no open ports on the router
- **Ubuntu Server** manages all hardware, networking, and storage at the OS level

### Storage Layout

```
/dev/nvme0n1  →  256 GB NVMe SSD (OS Drive)
                 ├── /boot/efi
                 ├── /boot
                 └── /  (Ubuntu + Docker + CasaOS)

/dev/sda      →  ~916 GB HDD (Data Drive at /mnt/data)
                 ├── /mnt/data/nextcloud/
                 │    ├── data/      ← User files
                 │    ├── config/    ← Nextcloud config
                 │    └── db/        ← PostgreSQL database
                 ├── /mnt/data/syncthing/
                 │    └── phone-photos/
                 └── /mnt/data/backups/
```

---

## Key Features

- **Full data sovereignty** — all files on owned hardware, no third-party access
- **Zero open router ports** — Tailscale VPN eliminates the entire port-forwarding attack surface
- **Multi-user access** with strict privacy separation between accounts
- **Automatic photo/video backup** from Android phones via Nextcloud app
- **Proactive HDD health monitoring** via S.M.A.R.T. with continuous daemon
- **Auto-start on boot** — all services come up without manual intervention
- **Scheduled daily shutdown** via cron (configurable)
- **Local LAN speeds of 50–100 MB/s** vs ~3–4 MB/s on cloud

---

## Quick Access Reference

| Service | Local (Home Network) | Remote (Tailscale VPN) |
|---|---|---|
| CasaOS Dashboard | `http://<server-local-ip>` | `http://<tailscale-ip>` |
| Nextcloud | `http://<server-local-ip>:7581` | `http://<tailscale-ip>:7581` |
| SSH | `ssh <user>@<server-local-ip>` | `ssh <user>@<tailscale-ip>` |

---

## Setup Phases (Summary)

The full setup is documented in [`NAS_Project_Documentation_Algorithm v1.2 (2).pdf`](./NAS_Project_Documentation_Algorithm v1.2 (2).pdf), but here's the high-level order:

1. **OS Installation** — Ubuntu Server 22.04 LTS via Rufus bootable USB
2. **Static IP Configuration** — Netplan config to prevent DHCP address changes
3. **HDD Preparation** — Format to ext4, mount at `/mnt/data`, persist via `fstab` with `nofail`
4. **CasaOS Installation** — Home server dashboard and Docker runtime
5. **Nextcloud Deployment** — Deployed via CasaOS custom install with volume mapped to `/mnt/data/nextcloud`
6. **Tailscale VPN Setup** — Installed, configured as exit node, key expiry disabled
7. **S.M.A.R.T. Monitoring** — Daemon registered for continuous HDD health checks
8. **Auto-Start & Scheduled Shutdown** — All services enabled on boot; 11 PM auto-shutdown via cron

---

## Challenges & Resolutions

Real issues hit during the build — documented for anyone following a similar path:

| # | Issue | Cause | Fix |
|---|---|---|---|
| 1 | System drops into emergency mode on boot | HDD was NTFS-formatted; fstab entry assumed ext4 | Reformatted HDD as ext4, corrected fstab entry with `nofail` |
| 2 | "Access through untrusted domain" on Nextcloud | IPs not listed in `trusted_domains` config | Added LAN IP, Tailscale IP, and port variants via `occ` CLI |
| 3 | Port conflict on 7580 | Default port flagged by CasaOS validation | Remapped Nextcloud container to port 7581 |
| 4 | Docker volume path error during install | PostgreSQL path is `/var/lib/postgresql/data`, not MySQL path | Corrected volume path in CasaOS custom install dialog |
| 5 | Brute force lockout on Nextcloud login | Repeated failed attempts during setup triggered throttle | Reset with `occ security:bruteforce:reset`, re-enabled protection after stable |
| 6 | IPv4 forwarding not activating | Typo `irtpv4` instead of `net.ipv4` in `sysctl.conf` | Corrected entry, applied with `sysctl -p` |
| 7 | ICRC errors in HDD S.M.A.R.T. log | Loose SATA data cable | Reseated cable; no reallocated sectors — drive assessed healthy |
| 8 | `occ` command: "Could not open input file" | Missing full path to `occ` binary inside container | Used full path: `php /var/www/html/occ` |

---

## Risks & Critical Rules

### ⚠️ Non-Negotiable Operating Rules

1. **Never cut power while the HDD is active** — always shut down via CasaOS dashboard or `sudo shutdown now`
2. **Run monthly external HDD backup** — keep the drive physically disconnected after each backup
3. **Check S.M.A.R.T. health monthly** — act immediately if Reallocated Sectors > 0
4. **Keep system updated weekly** — `sudo apt update && sudo apt upgrade -y`
5. **Never store files exclusively on the NAS** — always maintain at least one other copy
6. **No open router ports** — all remote access via Tailscale only

### Hardware Risk Summary

| Risk | Severity | Mitigation |
|---|---|---|
| Aging consumer HDD failure | **HIGH** | Monthly backup + SMART monitoring; plan replacement in 1–2 years |
| Power cut during HDD write | **HIGH** | UPS/inverter required; BIOS auto-start configured |
| SATA interface / ICRC errors | MEDIUM | Inspect and reseat cable; replace if errors persist |
| NVMe SSD wear | LOW | OS-only workload; monitor with `nvme-cli` |

---

## Backup Strategy (3-2-1 Rule)

| Copy | Type | Location |
|---|---|---|
| 1 | Primary (Live) | Toshiba HDD at `/mnt/data` — active working copy |
| 2 | Local Backup | Monthly `rsync` to external USB HDD — stored separately, disconnected after backup |
| 3 | Off-site *(planned)* | Secondary external HDD at separate location, or selective upload to free-tier cloud (Backblaze B2) |

---

## Future Upgrade Roadmap

| Upgrade | Priority | Notes |
|---|---|---|
| Replace HDD with NAS-rated drive (WD Red Plus / Seagate IronWolf) | **HIGH** | Current drive is consumer-grade and aging |
| RAID-1 mirroring via `mdadm` | **HIGH** | Real-time redundancy; single drive failure = zero data loss |
| Gigabit Ethernet (wired) | MEDIUM | Replace Wi-Fi for consistent LAN speeds |
| nginx Reverse Proxy + Let's Encrypt SSL | MEDIUM | HTTPS access, eliminates browser security warnings |
| Uptime Kuma monitoring dashboard | LOW | Visual service health status page via CasaOS |
| 24/7 operation | LOW (post-HDD upgrade) | After NAS-rated HDD is installed |

---

## Full Documentation

The complete technical documentation — including all setup algorithms, detailed configurations, user/privacy model, and operational procedures — is in:

📄 [`NAS_Project_Documentation_Algorithm v1.2 (2).pdf`](./NAS_Project_Documentation_Algorithm v1.2 (2).pdf)

---

*Built with open-source software. All configurations are generic and adaptable to similar hardware setups.*
