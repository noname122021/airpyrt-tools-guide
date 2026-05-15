# AirPort Time Capsule HDD Spindown (Sleep) Guide

This guide explains how to adjust the hard drive spindown (sleep) timer on an **AirPort Time Capsule (802.11ac / 5th Generation) - Model A1470**.

> [!IMPORTANT]
> **Experimental Status**: This method has not been tested by the repository maintainers as we do not have a Time Capsule on hand. It is based on the underlying NetBSD architecture of the device. Use at your own risk.
> 
> For discussion or to report results, please see **[Issue #1](https://github.com/noname122021/airpyrt-tools-guide/issues/1)**.

## 1. Overview
The Time Capsule does not have a user-accessible setting for HDD spindown. The drive is managed by the internal OS (NetBSD 6.0). By gaining root SSH access, you can use the `atactl` utility to set a custom inactivity timer.

## 2. Prerequisites
1.  **Root SSH Access**: You must first jailbreak your device. Follow the **[Setup Guide](MACOS_SILICON_GUIDE.md)** to:
    *   Install Python 3 AirPyrt tools.
    *   Enable the `dbug` flag (`0x3000`).
    *   Reboot the device.

## 3. Implementation

### Step 1: Connect via SSH
Connect to your Time Capsule as root:
```bash
ssh root@<ROUTER_IP>
```

### Step 2: Identify the Drive
Typically, the internal hard drive is mapped to `wd0`. You can verify this by running:
```bash
atactl wd0 identify
```

### Step 3: Set Spindown Timer
Use the `setstandby` command followed by the timeout in **seconds**.

*   **Set to 10 minutes (600s):**
    ```bash
    atactl wd0 setstandby 600
    ```
*   **Disable spindown (Always on):**
    ```bash
    atactl wd0 setstandby 0
    ```
*   **Check power state:**
    ```bash
    atactl wd0 checkpower
    ```

## 4. Making it Persistent
The AirPort filesystem is read-only and resets on reboot. To make this setting permanent, use a **USB Startup Script**.

1.  Connect a USB drive formatted as HFS+ to the Time Capsule.
2.  Follow **[Part 8 of the Setup Guide](MACOS_SILICON_GUIDE.md#part-8-usb-startup-scripts-automation)** to enable startup scripts.
3.  Add the `atactl` command to your `AirPort-startup.sh` script:
    ```bash
    #!/bin/sh
    # Set HDD spindown to 15 minutes
    /sbin/atactl wd0 setstandby 900
    ```

---

## 5. Safety & Considerations
*   **Mechanical Wear**: Avoid setting a very short timer (e.g., under 5 minutes), as frequent spin-up/spin-down cycles can damage the drive motor over time.
*   **Heat**: If you disable spindown (`0`), monitor the internal temperature using `envstat` to ensure the device stays within safe limits.
