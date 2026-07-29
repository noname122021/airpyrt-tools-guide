# AirPort Time Capsule HDD Spindown (Sleep) Guide

This guide explains how to adjust the hard drive spindown (sleep) timer on an **AirPort Time Capsule (802.11ac / 5th Generation) - Model A1470**.

> [!NOTE]
> **Status: Verified Working**  
> This feature has been physically tested and confirmed working on an AirPort Time Capsule A1470 running firmware 7.9.1 (NetBSD 6.0 evbarm). Special thanks to **[@gRiverOS](https://github.com/noname122021/airpyrt-tools-guide/issues/1)** for testing and reporting detailed hardware behavior.

---

## 1. Overview
The AirPort Time Capsule does not offer a user-accessible setting for HDD spindown in AirPort Utility. Drive power management is controlled by the internal OS (NetBSD 6.0). By gaining root SSH access, you can use the `atactl` utility to set custom inactivity sleep timers or manage drive state.

---

## 2. Prerequisites
1. **Root SSH Access**: You must jailbreak your device first. Follow the **[Setup Guide](MACOS_SILICON_GUIDE.md)** to:
   - Install Python 3 AirPyrt tools.
   - Enable the `dbug` flag (`0x3000`).
   - Reboot the device and connect via SSH (`ssh root@<ROUTER_IP>`).

---

## 3. Implementation

### Step 1: Connect via SSH
Connect to your Time Capsule as root:
```bash
ssh root@<ROUTER_IP>
```

### Step 2: Unmount Disk & Kill File Sharing (`afpserver`)
> [!IMPORTANT]
> **Critical Blocker**: `atactl` commands will fail with `atactl: /dev/wd0: Device busy` if the drive volume (`/Volumes/dk2`) is currently mounted.

The internal disk auto-mounts on demand whenever AFP or SMB file sharing is accessed. Standard `umount` will fail (`Device busy`) because `afpserver` keeps file descriptors open:

```bash
# fstat /Volumes/dk2
USER   CMD        PID  FD MOUNT        INUM MODE       SZ|DV R/W NAME
root   afpserver  501  15 /Volumes/dk2    2 drwxr-xr-x     0 r   Disk
```

To release the volume:
1. Kill `afpserver` to close active descriptors:
   ```bash
   killall afpserver
   ```
2. `diskd` will then unmount the disk volume automatically.

> [!NOTE]
> Killing `afpserver` disables network file sharing until the router is rebooted. A reboot takes under a minute and restores all services, as the root operating system image is rebuilt on every boot.

### Step 3: Identify the Drive
Verify that the internal hard drive is recognized as `wd0`:
```bash
atactl wd0 identify
```
*(Both `atactl wd0 identify` and `atactl /dev/wd0 identify` work identically).*

### Step 4: Set Spindown Timer
Use `setstandby` followed by the desired timeout in **seconds**:

* **Set to 10 minutes (600s):**
  ```bash
  atactl wd0 setstandby 600
  ```
* **Set to 1 minute (60s - useful for testing):**
  ```bash
  atactl wd0 setstandby 60
  ```
* **Disable spindown (Keep disk always spinning):**
  ```bash
  atactl wd0 setstandby 0
  ```
* **Force immediate spindown (Standby mode):**
  ```bash
  atactl wd0 standby
  ```

---

## 4. Verifying Spindown (Physical Verification Only)

> [!WARNING]
> **`atactl wd0 checkpower` is UNRELIABLE!**  
> Running `atactl wd0 checkpower` will return `Current power status: Active mode` even when the drive was successfully sleeping. Opening `/dev/wd0` causes NetBSD to read the disk label, which instantly wakes the drive before measuring its status.

### How to Properly Verify:
1. Set a short spindown timer (e.g., `atactl wd0 setstandby 60`).
2. Do **NOT** run any further `atactl` commands or access disk shares.
3. Perform a **physical check** after the timeout: touch the casing or listen closely.
   - When spun down: No vibration and no platter spin noise.

---

## 5. Persistence & Filesystem Architecture

The root filesystem (`/`) is not physically read-only, but it is a **RAM disk image (`tmpfs` / `md0a`)** that is rebuilt from firmware on every boot. Any changes made directly to `/` reset upon reboot.

However, the flash memory partition at `/mnt/Flash` (`/dev/flash2a`) **is persistent across reboots** (storing configuration binaries, SSH host keys, etc.).

### USB Startup Script (Recommended)
To automatically set the spindown timer on every reboot:
1. Format a USB flash drive as HFS+ and connect it to the Time Capsule.
2. Follow **[Part 8 of the Setup Guide](MACOS_SILICON_GUIDE.md#part-8-usb-startup-scripts-automation)** to enable startup scripts.
3. Add the spindown commands to your `AirPort-startup.sh`:
   ```bash
   #!/bin/sh
   # Unmount internal storage to allow atactl execution
   killall afpserver

   # Set HDD spindown to 15 minutes (900 seconds)
   /sbin/atactl wd0 setstandby 900
   ```

---

## 6. SMART Health Diagnostics

Because Time Capsules contain mechanical drives that are now over a decade old, it is recommended to check drive health before relying on it for backups.

Run SMART health status:
```bash
atactl wd0 smart status
```

**Example Output:**
```
  5 Reallocated sector count   0
197 Current pending sector     0
198 Offline uncorrectable      0
194 Temperature                29 C
  9 Power-on hours             9945
  4 Start/stop count           13370
193 Load cycle count           13717
```

Key indicators:
- **Reallocated / Pending Sectors**: Values above `0` indicate failing storage media.
- **Temperature**: Typical operational range is 25°C – 45°C.

---

## 7. Safety & Considerations

* **Mechanical Wear**: Avoid setting an excessively short spindown timer (e.g., under 5 minutes / 300s) for daily use. Frequent spin-up and spin-down cycles increase mechanical stress on motor bearings and heads, accelerating drive degradation—especially on 10+ year old mechanical HDDs. A value of 10 to 15 minutes (600s – 900s) is recommended.
* **Thermal Management**: If you disable spindown (`atactl wd0 setstandby 0`), monitor the internal drive temperature using `atactl wd0 smart status` or `envstat` to ensure thermal limits remain safe inside the compact enclosure.

---

## 8. Reference: Available `atactl` Commands

The NetBSD 6.0 build on the AirPort Time Capsule supports the following `atactl` subcommands:

| Command | Description |
| :--- | :--- |
| `identify` | Display drive model, revision, and serial number |
| `setidle <seconds>` | Set drive idle timer |
| `setstandby <seconds>` | Set drive standby (spindown) timer |
| `idle` | Put drive into idle mode |
| `standby` | Put drive into standby mode (spin down immediately) |
| `sleep` | Put drive into sleep mode |
| `checkpower` | Query power status *(Note: wakes drive on call)* |
| `apm disable\|set #` | Advanced Power Management control |
| `smart status` | View SMART attributes and drive health |
| `smart enable\|disable` | Enable or disable SMART logging |
| `security status` | Check ATA security lock state |

*Note: `getparam` is not compiled into this firmware build, so reading back the active timer value directly via CLI is unsupported.*

---

## 9. Test Environment & Credits

- **Tested Device**: AirPort Time Capsule 802.11ac (Model A1470)
- **Firmware**: 7.9.1 (`AirPortFW-79100.2`, build 2019-04-29)
- **Architecture**: NetBSD 6.0 (`evbarm`, target J28)
- **Internal Drive**: APPLE HDD ST3000DM001 (3 TB, Rev AP54)
- **Contributor**: Hardware testing and detailed issue breakdown provided by **[@gRiverOS](https://github.com/noname122021/airpyrt-tools-guide/issues/1)**.
