# Suricata Configuration Troubleshooting Guide

*Resolving systemd service override conflicts and interface configuration issues*

---

## Problem Overview

### Symptoms

- Suricata starts successfully but monitors the wrong network interface
- Configuration file changes to interface setting are ignored
- No threat signatures loaded (0 rules)
- No log files created in configured log directory
- Logs show `interface: eth0` despite configuring `wlan1` (or your intended interface)

### Root Cause

Suricata has two configuration layers that can conflict:

**1. Configuration File (`/etc/suricata/suricata.yaml`)**
Contains detailed settings including interface selection, log directories, and rule paths.

**2. Systemd Service File (`/usr/lib/systemd/system/suricata.service`)**
Contains the `ExecStart` command that launches Suricata with command-line arguments.

**Priority order:** Command-line arguments override configuration file settings.

When the systemd service specifies `--af-packet` without an interface, Suricata defaults to the first available interface (usually `eth0`), **overriding the configuration file setting.**

---

## Diagnostic Steps

### Step 1: Verify Configuration File

```bash
grep "interface:" /etc/suricata/suricata.yaml
```

Expected output: `- interface: wlan1` (or your intended interface).

### Step 2: Check Which Interface Suricata Is Actually Using

```bash
sudo cat /var/log/suricata/suricata.log | grep -E 'wlan|eth0'
```

**Problem indicators:**
- `eth0: creating 4 threads` — wrong interface
- `eth0: packets: 0` — no traffic captured

**Success indicators:**
- `wlan1: creating 4 threads` — correct interface
- `wlan1: packets: 12239` — traffic being captured

### Step 3: Inspect Systemd Service

```bash
sudo systemctl cat suricata
```

Look for the `ExecStart` line. If it shows:
```
ExecStart=/usr/bin/suricata -D --af-packet -c /etc/suricata/suricata.yaml --pidfile /run/suricata.pid
```

The `--af-packet` without an interface name is the problem — Suricata will choose the default interface.

### Step 4: Check for Missing Rules

```bash
sudo cat /var/log/suricata/suricata.log | grep "rule"
```

**Problem indicators:**
```
Warning: detect: No rule files match the pattern
Info: detect: 0 signatures processed
```

This means `suricata-update` failed or rules weren't installed correctly.

---

## Solution: Create a Systemd Override

The correct fix is a systemd override that explicitly specifies the monitoring interface. This preserves the original service file and survives package upgrades.

### Step 1: Create Override

```bash
sudo systemctl edit suricata
```

This creates `/etc/systemd/system/suricata.service.d/override.conf`.

### Step 2: Add Override Configuration

```ini
[Service]
ExecStart=
ExecStart=/usr/bin/suricata -D --af-packet=wlan1 -c /etc/suricata/suricata.yaml --pidfile /run/suricata.pid
```

> Replace `wlan1` with your actual monitoring interface.

**Why two `ExecStart` lines?**
- `ExecStart=` (empty) clears the original command — required by systemd before redefining
- `ExecStart=/usr/bin/suricata...` sets the new command with the interface explicitly specified

### Step 3: Reload and Restart

```bash
sudo systemctl daemon-reload
sudo systemctl restart suricata
```

---

## Verification

### Check Status

```bash
sudo systemctl status suricata
```

Expected: `Active: active (running)`

### Verify Interface

```bash
sudo tail -100 /var/log/suricata/suricata.log | grep wlan1
```

Expected:
```
Info: runmodes: wlan1: creating 4 threads
Notice: device: wlan1: packets: 12239, drops: 0
```

### Verify Rules Loaded

```bash
sudo tail -100 /var/log/suricata/suricata.log | grep signatures
```

Expected:
```
Info: detect: 48549 signatures processed...
```

---

## Additional Issues

### YAML Syntax Error — Variable Not Defined

**Symptom:**
```
Error: Variable " HOME_NET" is not defined
```

**Cause:** Line break or extra space in a YAML variable reference.

**Example of broken config:**
```yaml
MODBUS_CLIENT: "$
HOME_NET"
```

**Fix:** Edit `/etc/suricata/suricata.yaml` and join the broken line:
```yaml
MODBUS_CLIENT: "$HOME_NET"
```

### Missing Suricata User Account

**Symptom:**
```
chown: invalid user: 'suricata:suricata'
```

**Fix:**
```bash
sudo useradd -r -M -s /usr/sbin/nologin suricata
```

- `-r` — system account
- `-M` — no home directory
- `-s /usr/sbin/nologin` — no shell access

### No Log Files Created

**Possible causes:**
1. Suricata monitoring wrong interface (no traffic to log)
2. Permission denied on log directory
3. Logs being written to default location instead of configured path

**Check default log location:**
```bash
ls -lh /var/log/suricata/
```

If logs exist here, the systemd override hasn't been applied yet. Suricata is using the default path instead of your configured one.

**Check permissions:**
```bash
ls -ld /path/to/your/log/directory/
```

Should show `suricata` as owner.

---

## Suricata Log Files Reference

| Log File | Contents |
|----------|----------|
| `fast.log` | Quick alert summary — one line per alert with timestamp, signature, and IPs |
| `eve.json` | Detailed JSON logs — alerts, flows, DNS, HTTP, SSL. Used for SIEM forwarding |
| `stats.log` | Performance stats — packet rates, drops, memory usage |
| `suricata.log` | Startup messages, config parsing, rule loading, operational info |

---

## Prevention

On fresh Suricata installations:
1. Install Suricata
2. Immediately create systemd override with explicit interface
3. Edit `/etc/suricata/suricata.yaml` for other settings
4. Run `suricata-update` to install rules
5. Create `suricata` user if needed
6. Set log directory permissions
7. Start service and verify in logs

---

## Quick Reference

```bash
# Create systemd override
sudo systemctl edit suricata

# Reload and restart
sudo systemctl daemon-reload
sudo systemctl restart suricata

# Verify configuration
sudo systemctl cat suricata
sudo systemctl status suricata
sudo tail -100 /var/log/suricata/suricata.log
```
