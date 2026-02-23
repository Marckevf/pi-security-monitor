# Suricata IDS — Configuration & Troubleshooting Guide

Suricata is an open-source intrusion detection system (IDS) that inspects network traffic against a library of threat signatures. This guide covers common configuration issues encountered when deploying Suricata on a Raspberry Pi and how to resolve them.

---

## Understanding Suricata's Configuration Layers

Suricata reads settings from two places, and they can conflict:

**1. `/etc/suricata/suricata.yaml`** — the main config file. Controls interface selection, log paths, rule locations, and output formats.

**2. The systemd service file** — controls how Suricata is launched. Command-line arguments here take priority over the config file.

This priority order is the source of most configuration issues. If the systemd service launches Suricata with `--af-packet` and no interface specified, Suricata will default to the first available interface (usually `eth0`) regardless of what's set in `suricata.yaml`.

---

## Issue 1: Suricata Monitoring the Wrong Interface

### Symptoms
- Suricata starts without errors but captures no traffic
- Logs reference the wrong interface
- No alerts generated despite active network traffic

### Diagnosis

Check which interface Suricata is actually using:

```bash
sudo grep -i "interface\|creating.*threads" /var/log/suricata/suricata.log
```

Inspect the systemd service to see how Suricata is being launched:

```bash
sudo systemctl cat suricata
```

If the `ExecStart` line looks like this:
```
ExecStart=/usr/bin/suricata -D --af-packet -c /etc/suricata/suricata.yaml
```

The `--af-packet` flag without an interface name is the problem.

### Solution: Create a Systemd Override

Rather than editing the original service file (which gets overwritten on package upgrades), create an override:

```bash
sudo systemctl edit suricata
```

Add the following:

```ini
[Service]
ExecStart=
ExecStart=/usr/bin/suricata -D --af-packet=YOUR_INTERFACE -c /etc/suricata/suricata.yaml --pidfile /run/suricata.pid
```

Replace `YOUR_INTERFACE` with your monitoring interface (e.g. `wlan1`, `eth0`, `enp3s0`).

> **Why two `ExecStart` lines?** The first empty line clears the original command — systemd requires this before redefining it. The second line sets the new command with the interface explicitly specified.

Apply the changes:

```bash
sudo systemctl daemon-reload
sudo systemctl restart suricata
```

### Verification

```bash
sudo systemctl status suricata
sudo grep "YOUR_INTERFACE" /var/log/suricata/suricata.log
```

You should see threads created on your interface and packets being captured.

---

## Issue 2: No Rules Loaded

### Symptoms
- Suricata starts but generates no alerts
- Logs show `0 signatures processed`

### Diagnosis

```bash
sudo grep "signatures" /var/log/suricata/suricata.log
```

### Solution

Run the rule updater and restart:

```bash
sudo suricata-update
sudo systemctl restart suricata
```

Verify rules loaded:

```bash
sudo grep "signatures processed" /var/log/suricata/suricata.log
```

A healthy install typically shows 40,000+ signatures loaded.

---

## Issue 3: No Log Files Created

### Symptoms
- Suricata is running but the configured log directory is empty
- Logs are appearing in `/var/log/suricata/` instead of your custom path

### Cause

If Suricata is writing to the default location rather than your configured path, the systemd override likely hasn't been applied yet, or the log directory has incorrect permissions.

### Solution

Verify your log directory exists with correct ownership:

```bash
sudo mkdir -p /your/log/directory
sudo chown -R suricata:suricata /your/log/directory
```

If the `suricata` system user doesn't exist:

```bash
sudo useradd -r -M -s /usr/sbin/nologin suricata
```

Restart Suricata:

```bash
sudo systemctl restart suricata
```

---

## Log Files Reference

| File | Contents |
|------|----------|
| `eve.json` | Full JSON output — alerts, flows, DNS, HTTP, SSL. Primary file for SIEM forwarding |
| `fast.log` | One-line alert summaries with timestamp, signature name, and IPs |
| `stats.log` | Performance metrics — packet rates, dropped packets, memory usage |
| `suricata.log` | Startup output, config parsing, rule loading, and operational messages |

---

## Quick Reference

```bash
# Create or edit systemd override
sudo systemctl edit suricata

# Apply changes
sudo systemctl daemon-reload && sudo systemctl restart suricata

# Check status
sudo systemctl status suricata

# View logs
sudo tail -50 /var/log/suricata/suricata.log

# Update rules
sudo suricata-update
```
