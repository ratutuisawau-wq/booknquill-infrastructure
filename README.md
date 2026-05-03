# BookNQuill SMP — Multi-Region Minecraft Server Infrastructure

A globally-distributed Minecraft community server using an edge-proxy / origin-isolation architecture. Built and maintained solo as a personal infrastructure project.

## TL;DR

Three regional Velocity proxies (Dallas, Frankfurt, Singapore) accept player connections from the nearest geographic region via AWS Route 53 geolocation routing. All proxies forward traffic over a private WireGuard VPN mesh to a single hardened Paper backend that has **no public exposure on the Minecraft application port** — its host firewall only accepts connections from the four authorized WireGuard peer IPs.

This is the same edge-proxy / origin-isolation pattern used in production web infrastructure, applied to a Minecraft community server.

## Architecture

```
                    ┌──────────────────────────────────────────┐
                    │  AWS Route 53 (geolocation routing)      │
                    │  booknquillsmp.net + bedrock.<...>:19132 │
                    └─────┬───────────────┬───────────────┬────┘
                          │               │               │
                  ┌───────▼─────┐ ┌───────▼─────┐ ┌───────▼─────┐
                  │  Velocity   │ │  Velocity   │ │  Velocity   │
                  │   Dallas    │ │  Frankfurt  │ │  Singapore  │
                  │ + Geyser/   │ │ + Geyser/   │ │ + Geyser/   │
                  │  Floodgate  │ │  Floodgate  │ │  Floodgate  │
                  └───────┬─────┘ └───────┬─────┘ └───────┬─────┘
                          │               │               │
                          └───────┬───────┴───────┬───────┘
                                  │  WireGuard    │
                                  │  VPN mesh     │
                                  │               │
                          ┌───────▼───────────────▼───────┐
                          │     Paper Backend (Vultr)     │
                          │  — no public exposure on      │
                          │    the application port —     │
                          │  UFW: peer IPs only           │
                          └───────────────────────────────┘
```

## Goals

- **Low-latency global access** — players land on the nearest regional proxy, not a single distant server
- **Single backend, single world** — Java and Bedrock players share the same world, the same economy, the same player base
- **Hardened backend** — application port is unreachable from the public internet; only authorized VPN peers can connect
- **Operable solo** — auto-start on boot, scheduled restarts with player notifications, full runbook documentation

## Tech Stack

| Layer | Technology |
| --- | --- |
| DNS / routing | AWS Route 53 (geolocation policy) |
| Proxy | Velocity (3× regional Vultr VPS) |
| Java/Bedrock unification | Geyser + Floodgate |
| Backend | Paper 1.21.x on Vultr (4 vCPU / 8GB / 180GB) |
| Private network | WireGuard VPN mesh (5 nodes) |
| OS | Ubuntu 22.04 LTS |
| Service mgmt | systemd, cron, screen, UFW/iptables |
| Voice | Simple Voice Chat (direct UDP) |

## Why This Pattern

Running Paper on a single public VPS is simpler. So why this?

- **DDoS attacks hit the proxy layer**, not the backend that holds player data and the world file. Proxies are stateless and replaceable; the backend is not.
- **Backend can be moved or replaced** with a single config change at the proxy layer — no DNS propagation delay, no player-visible downtime beyond a restart.
- **Initial connection latency is reduced** because the player's TCP handshake completes against a regional proxy instead of a transcontinental link.
- **Backend attack surface is reduced to SSH and the WireGuard listen port.** Port scans against the backend's public IP show the application port closed.

## Notable Operational Details

- **Aikar's Flags** for JVM tuning — substantially reduces garbage-collection pauses vs. defaults on Paper
- **Daily scheduled restart** via cron with 60s + 10s in-game broadcast warnings and graceful shutdown
- **WireGuard** authenticates by keypair, not source IP — peers can roam without breaking the mesh
- **Geyser/Floodgate on the proxy layer** means Java and Bedrock players share one world without dual-stacking the backend
- **UFW + persistent iptables** rules survive reboot via netfilter-persistent

## Latency (Backend → Region)

| Path | RTT |
| --- | --- |
| Backend → Dallas proxy (same DC) | ~1 ms |
| Backend → Frankfurt proxy | ~120 ms |
| Backend → Singapore proxy | ~210 ms |

Note: this is the proxy ↔ backend leg, not what the player sees. Players experience the latency to their nearest proxy (typically 10–40 ms within their continent) plus the proxy → backend leg, but only the latter is the cross-continent hop.

## Architecture Evolution

This is not the first version of the system. Earlier iterations used:

1. **Home PC as backend** — migrated off because residential ISP upload bandwidth and reliability did not scale, and exposing a residential network to public Minecraft traffic was a security concern.
2. **playit.gg tunneling for voice chat** — removed once voice chat was consolidated onto the Vultr backend; eliminated a third-party dependency and reduced latency.
3. **Manual per-region subdomains** — replaced with Route 53 geolocation policies that route automatically based on the resolver's location.

Each migration either reduced operational complexity, lowered latency, or tightened the network boundary around the backend.

## Lessons Learned

- **Backend isolation is worth the operational cost.** Initial setup is more involved than a single public VPS, but the security and operational flexibility benefits compound.
- **WireGuard is the right tool for proxy-to-backend trust.** Keypair-authenticated, encrypted, and tolerant of intermittent connectivity. Five-node mesh setup is straightforward; ongoing maintenance is essentially zero.
- **Documentation is part of the system.** Operating a multi-server stack solo without runbooks is unsustainable. A single source of truth for configuration, file paths, and procedures pays for itself during any incident.
- **Credentials never live in shareable documents.** This README is the public version. The operational version with IPs, keys, and forwarding secrets is maintained separately.

## Full Documentation

Full architecture document available in this repo: [`BookNQuill_SMP_Architecture.docx`](./BookNQuill_SMP_Architecture.docx)

## About

Built and maintained by [Ratu Tuisawau](https://github.com/ratutuisawau-wq). Sacramento, CA. IT support technician studying for CompTIA Network+ and Security+, currently building toward a career in cybersecurity. Operates [SacManaged Tech](https://sacmanagedtech.com).
