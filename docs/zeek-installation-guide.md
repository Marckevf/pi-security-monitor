# Zeek Network Security Monitor — Installation & Configuration Guide

*Raspberry Pi Security Monitoring Project — Phase 2: Behavioral Analysis & Network Logging*

---

## Overview

Zeek (formerly Bro) is an open-source passive network analysis framework that inspects all passing traffic. Unlike signature-based IDS tools like Suricata, Zeek generates structured, detailed logs of all network activity — providing deep behavioral visibility into what is happening on your network.

In this project, Zeek is deployed on a Raspberry Pi acting as a wireless access point, monitoring all traffic from connected client devices in real-time.

**What Zeek logs:**
- Every connection (source, destination, protocol, duration, bytes)
- DNS queries and responses
- HTTP requests, responses, and headers
- SSL/TLS certificate details
- DHCP transactions and IP assignments
- SSH sessions
- Protocol anomalies and suspicious behavior

---

## System Requirements

| Requirement | Details |
|-------------|---------|
| Platform | Raspberry Pi 4 (2GB+ RAM recommended) |
| OS | Raspberry Pi OS (Debian Bullseye/Bookworm or later) |
| Interface | Dedicated interface for monitoring AP traffic (e.g. `wlan1`) |
| Storage | USB drive recommended — logs can be large |
| Disk Space | ~500MB for Zeek install; log volume varies by traffic |

---

## Installation

### Step 1: Update System

```bash
sudo apt update && sudo apt upgrade -y
```

### Step 2: Install Zeek

```bash
sudo apt install zeek -y
```

This installs Zeek along with `libpcap` for packet capture.

### Step 3: Verify Installation

```bash
zeek --version
zeekctl --version
```

### Step 4: Add Zeek to PATH

```bash
echo 'export PATH=$PATH:/opt/zeek/bin' >> ~/.bashrc
source ~/.bashrc
```

---

## Configuration

Configuration files are in `/etc/zeek/` (or `/opt/zeek/etc/` depending on installation method).

### Step 5: Set Monitoring Interface

Edit `node.cfg`:

```bash
sudo nano /etc/zeek/node.cfg
```

```ini
[zeek]
type=standalone
host=localhost
interface=wlan1    # Change to your monitoring interface
```

### Step 6: Define Local Networks

Edit `networks.cfg` to tell Zeek which addresses are internal:

```bash
sudo nano /etc/zeek/networks.cfg
```

```
10.0.0.0/8          Private network
192.168.0.0/16      Private network
172.16.0.0/12       Private network
```

Add your specific AP subnet too (e.g., `10.3.141.0/24`).

### Step 7: Set Log Directory

Edit `zeekctl.cfg` to redirect logs away from the default location:

```bash
sudo nano /etc/zeek/zeekctl.cfg
```

```ini
LogDir = /mnt/security-logs/zeek
```

Create the directory:

```bash
sudo mkdir -p /mnt/security-logs/zeek
sudo chown YOUR_USERNAME:YOUR_USERNAME /mnt/security-logs/zeek
```

### Step 8: Enable JSON Output

Edit `site/local.zeek`:

```bash
sudo nano /etc/zeek/site/local.zeek
```

Add at the bottom:

```zeek
@load tuning/json-logs
```

This outputs all logs as JSON instead of TSV — required for Fluent Bit and Elasticsearch ingestion.

---

## Starting Zeek

### Deploy and Start

```bash
sudo zeekctl deploy
```

`deploy` runs install, start, and check in sequence.

### Enable on Boot

```bash
sudo systemctl enable zeek
sudo systemctl is-enabled zeek
```

---

## ZeekControl Commands

| Command | Description |
|---------|-------------|
| `sudo zeekctl status` | Check if Zeek is running |
| `sudo zeekctl start` | Start Zeek |
| `sudo zeekctl stop` | Stop Zeek |
| `sudo zeekctl restart` | Restart (apply config changes) |
| `sudo zeekctl deploy` | Full redeploy (init + start + check) |
| `sudo zeekctl check` | Validate config without starting |
| `sudo zeekctl diag` | Show diagnostic info |

---

## Log Files Reference

Active logs are in `YOUR_LOG_DIR/current/`:

| Log File | Contents |
|----------|----------|
| `conn.log` | Every connection: src/dst IP, port, protocol, duration, bytes |
| `dns.log` | All DNS queries and responses |
| `http.log` | HTTP requests: method, URL, response code, user-agent |
| `ssl.log` | SSL/TLS sessions and certificate info |
| `dhcp.log` | DHCP leases: MAC address, hostname, assigned IP |
| `ssh.log` | SSH connections and auth outcomes |
| `weird.log` | Protocol anomalies and unexpected behavior |
| `notice.log` | Alerts and notable events from Zeek policy scripts |
| `files.log` | Files observed or extracted from network traffic |

### View Logs in Real-Time

```bash
# Watch DNS queries
sudo tail -f /mnt/security-logs/zeek/current/dns.log | python3 -m json.tool

# Watch all connections
sudo tail -f /mnt/security-logs/zeek/current/conn.log
```

### Search Historical (Compressed) Logs

```bash
# Search for a specific IP
sudo zcat /mnt/security-logs/zeek/2026-02-21/conn*.log.gz | grep "192.168.1.5"

# Find DNS queries for a domain
sudo zcat /mnt/security-logs/zeek/2026-02-21/dns*.log.gz | grep "example.com"
```

---

## Log Rotation

Zeek rotates logs hourly automatically. Configure retention in `zeekctl.cfg`:

```ini
LogRotationInterval = 3600    # Rotate every hour (seconds)
LogExpireInterval = 7         # Delete logs older than 7 days
```

Alternatively, use a cron job for deletion:

```bash
sudo crontab -e
```

Add:
```
0 2 * * * find /mnt/security-logs -type f -mtime +7 -delete
```

---

## Troubleshooting

### Zeek Won't Start

```bash
sudo zeekctl diag
```

Common causes:
- Wrong interface name in `node.cfg`
- Permission errors on log directory
- `libpcap` unable to open interface

Verify the interface exists:
```bash
ip link show wlan1
```

### No Logs Being Generated

Verify Zeek can see traffic:
```bash
sudo tcpdump -i wlan1 -c 10
```

If no packets appear, the interface may not be in promiscuous mode or traffic isn't being routed through it.

### High CPU Usage

Monitor Zeek CPU:
```bash
top -p $(pgrep zeek)
```

If consistently above 80%, reduce logging verbosity in `local.zeek` by commenting out less critical log streams.

---

## Integration with Suricata

Zeek and Suricata serve complementary roles:

| Tool | Role |
|------|------|
| Suricata | Signature-based IDS — alerts on known attack patterns |
| Zeek | Behavioral logging — records all network activity for analysis and forensics |

In Phase 3, both Suricata and Zeek logs are forwarded to Elasticsearch using Fluent Bit. See `elasticsearch-setup-guide.md` for details.
