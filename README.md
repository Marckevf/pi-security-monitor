# 🛡️ Raspberry Pi Network Security Monitoring Station

A portable, self-contained network security monitoring platform built on a Raspberry Pi 4. The system operates as a wireless access point while performing real-time intrusion detection and behavioral traffic analysis on all connected devices — transparently, without impacting clients.

---

## 📐 Architecture

```
Internet ←→ wlan0 (uplink) ←→ Raspberry Pi 4 ←→ wlan1 (AP) ←→ Client Devices
                                      │
                              ┌───────┴────────┐
                          Suricata           Zeek
                       (IDS Alerts)    (Full Protocol Logs)
                              └───────┬────────┘
                                 Fluent Bit
                               (Log Forwarder)
                                      │
                               [Tailscale VPN]
                                      │
                               Elasticsearch
                               (Mac - Docker)
                                      │
                                   Kibana
                              (Live Dashboard)
                         accessible from Mac + Windows
```

All traffic from connected devices passes through the Pi before reaching the internet. Suricata and Zeek inspect every packet in real-time with zero client impact. Logs are forwarded over a private Tailscale VPN to Elasticsearch running in Docker on a Mac.

---

## 🧰 Tools & Technologies

| Tool | Version | Role |
|------|---------|------|
| Raspberry Pi OS | Debian Trixie | Base operating system (runs from USB) |
| Suricata | 7.x | Signature-based intrusion detection (48,000+ rules) |
| Zeek | 6.x | Behavioral network analysis and protocol logging |
| RaspAP | Latest | Wireless access point management |
| Tailscale | Latest | Zero-config VPN for secure remote access |
| Fluent Bit | 4.x | Lightweight log forwarding agent |
| Elasticsearch | 9.x | Log storage and indexing (Docker on Mac) |
| Kibana | 9.x | Visualization and live dashboards (Docker on Mac) |
| Docker Desktop | Latest | Container runtime for Elasticsearch + Kibana |

---

## ✨ Features

- **Real-time threat detection** — Suricata monitors all traffic against 48,000+ signatures and fires alerts instantly
- **Full protocol logging** — Zeek logs every connection, DNS query, HTTP request, SSL certificate, DHCP lease, SSH session, and more
- **Centralized log forwarding** — Fluent Bit streams logs from the Pi to a Mac over Tailscale VPN
- **Live Kibana dashboard** — visualize traffic patterns, alerts, and protocol breakdowns in real-time
- **Multi-device access** — Kibana accessible from any device on the Tailscale network (Mac, Windows, etc.)
- **Secure remote management** — access the Pi from anywhere via Tailscale without port forwarding
- **Automated log rotation** — hourly compression and 7-day retention keeps storage healthy
- **Portable** — runs headless from USB, boots automatically, recovers from crashes without intervention

---

## 📁 Repository Structure

```
pi-security-monitor/
├── README.md
├── configs/
│   ├── suricata/
│   │   └── suricata.yaml              # Suricata main config
│   ├── zeek/
│   │   ├── node.cfg                   # Interface configuration
│   │   └── zeekctl.cfg                # ZeekControl settings
│   ├── fluent-bit/
│   │   └── fluent-bit.conf            # Log forwarding configuration
│   └── docker-compose.yml             # Elasticsearch + Kibana (ports bound to 0.0.0.0 for Tailscale)
└── docs/
    ├── zeek-installation-guide.md     # Zeek installation and configuration
    ├── elasticsearch-setup-guide.md   # Elasticsearch + Kibana + Fluent Bit setup
    ├── suricata-troubleshooting.md    # Suricata interface and systemd override fixes
    ├── tailscale-dns-troubleshooting.md  # Fixing DNS conflicts with Tailscale
    └── raspap-boot-timing-fix.md      # Resolving RaspAP boot loop race conditions
```

---

## 🚀 Project Phases

| Phase | Description | Status |
|-------|-------------|--------|
| Phase 1 | Base OS, wireless AP (RaspAP), VPN (Tailscale), NAT/routing | ✅ Complete |
| Phase 2 | Suricata IDS + Zeek NSM deployment and configuration | ✅ Complete |
| Phase 3 | Fluent Bit log forwarding to Elasticsearch over Tailscale | ✅ Complete |
| Phase 4 | End-to-end testing and validation | ✅ Complete |
| Phase 5B | Kibana dashboards — live visualization and alerting | ✅ Complete |

---

## 📊 Dashboard

Kibana dashboards built on live data:

- **Alerts Over Time** — bar chart of Suricata alert volume
- **Alert Categories** — pie chart of alert type breakdown
- **Top Destination IPs** — most contacted external IPs
- **Top Alert Signatures** — which Suricata rules fire most
- **Protocol Breakdown** — network protocol distribution
- **Top Source IPs** — most active devices on the network

---

## 🗂️ Log Locations (on Pi)

| Tool | Active Logs | Archived Logs |
|------|-------------|---------------|
| Suricata | `/mnt/security-logs/suricata/eve.json` | Same directory |
| Zeek | `/mnt/security-logs/zeek/current/` | `/mnt/security-logs/zeek/YYYY-MM-DD/` |

Zeek logs rotate hourly and compress automatically to `.gz` files. Suricata outputs structured JSON via the EVE format. All logs are automatically deleted after 7 days via cron to manage USB storage.

---

## 🔧 Setup Overview

**Pi Setup:**
1. Flash Raspberry Pi OS to USB drive and boot
2. Configure dual-WiFi with RaspAP (wlan0 uplink, wlan1 AP)
3. Set up Tailscale for remote access
4. Install and configure Suricata with ET Open ruleset
5. Install and configure Zeek (via `/opt/zeek/`)
6. Mount external storage to `/mnt/security-logs/`
7. Install Fluent Bit and configure log forwarding to Elasticsearch
8. Set up cron job for 7-day log retention

**Mac Setup:**
1. Install Docker Desktop
2. Run Elastic start-local script to deploy Elasticsearch + Kibana
3. Edit `docker-compose.yml` to bind ports to `0.0.0.0` (required for Tailscale access)
4. Restart containers and create Kibana data view for `security-logs*`

**Tailscale Note:** The `docker-compose.yml` in this repo has ports bound to `0.0.0.0` instead of `127.0.0.1`. This is required to make Kibana and Elasticsearch accessible from other Tailscale devices. Do not expose these ports to the public internet.

---

## 🎓 Skills Demonstrated

- Linux system administration (Debian, systemd, networking)
- Network security monitoring and IDS/IPS configuration
- Wireless access point setup and traffic routing
- Deep packet inspection and protocol analysis
- Log management, forwarding, and centralized aggregation
- VPN configuration and secure remote access (Tailscale)
- Docker containerization and service orchestration
- ELK stack deployment and configuration
- Data visualization and security dashboards (Kibana)
- Embedded systems and resource-constrained deployment

---

## ⚠️ Legal Notice

This project is built for educational purposes on a personally owned network. All monitoring is performed on personally owned networks for educational purposes only.

---

## 📄 License

MIT License — feel free to use this as a reference for your own security monitoring projects.
