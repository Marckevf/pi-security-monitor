# RaspAP Boot Loop Fix — Service Timing Race Condition

RaspAP manages a dual-WiFi setup where one adapter connects to an upstream network and another runs as an access point. Getting the startup order right is critical. This guide explains what causes boot loop errors and how to fix them.

---

## Understanding the Problem

RaspAP uses a service daemon called `raspapd` to control the order in which network services start. It stops all services, configures interfaces, then starts them back up in a specific sequence:

1. `hostapd` — creates the access point
2. `dhcpcd` — assigns IP addresses to interfaces
3. `dnsmasq` — serves DHCP to connected clients

Each service needs time to fully initialize before the next one starts. The default delay between each step is 1 second — which is not always enough, particularly when booting from USB or when WiFi adapters need extra time to switch modes.

When services start before interfaces are ready, they bind to the loopback interface (`lo`) instead. When the real interfaces come up, an IP conflict occurs and the cycle repeats — causing an infinite boot loop.

---

## Symptoms

The Pi displays repeating errors during boot:

```
lo ate my ip address
```

This prevents the system from reaching a login prompt and makes SSH inaccessible.

---

## The Fix

Increase the delay between service starts from 1 second to 5 seconds.

> **Note:** If your Pi is stuck in a boot loop, you cannot SSH in to apply this fix. You must mount the storage device from another Linux computer.

### Step 1: Power Off and Remove Storage

Unplug the Pi and remove the USB drive or SD card.

### Step 2: Connect to Another Linux Computer

Plug the drive into another Linux machine.

### Step 3: Identify the Drive

```bash
lsblk
```

Look for two partitions — a small boot partition (vfat) and a larger root partition (ext4). Note the device name (e.g. `sdb`, `sdc`).

### Step 4: Mount the Root Partition

```bash
sudo mkdir -p /mnt/pi-root
sudo mount /dev/sdb2 /mnt/pi-root
```

Replace `sdb2` with your actual root partition.

### Step 5: Edit the raspapd Service File

```bash
sudo nano /mnt/pi-root/usr/lib/systemd/system/raspapd.service
```

Find:
```
ExecStart=/bin/bash /etc/raspap/hostapd/servicestart.sh --seconds 1
```

Change to:
```
ExecStart=/bin/bash /etc/raspap/hostapd/servicestart.sh --seconds 5
```

Save: `Ctrl+X`, `Y`, Enter.

### Step 6: Unmount and Reinstall

```bash
sudo umount /mnt/pi-root
```

Plug the drive back into the Pi and power on. The boot loop should be resolved.

> Boot time will increase by approximately 20 seconds. This is expected and only affects startup — runtime performance is unchanged.

---

## Verification

After successful boot:

```bash
# Check interfaces are up
ip addr show

# Check services are running
sudo systemctl status hostapd
sudo systemctl status dnsmasq
sudo systemctl status dhcpcd
```

All three services should show `active (running)`.

---

## Choosing the Right Delay

5 seconds works for most setups. If the boot loop persists, try 7 or 10 seconds. Factors that may require a longer delay include slower USB drives, certain WiFi adapter drivers, and heavy system load during boot.

---

## Prevention

After a fresh RaspAP installation, increase the delay before configuring anything else:

1. Edit `/usr/lib/systemd/system/raspapd.service`
2. Change `--seconds 1` to `--seconds 5`
3. Run `sudo systemctl daemon-reload`
4. Test the boot a few times before proceeding with configuration
