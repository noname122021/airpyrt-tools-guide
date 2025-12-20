# AirPort Extreme 802.11ac Tools & Hardware Guide

This repository serves as a documentation hub and backup for tools related to the **Apple AirPort Extreme 802.11ac (Tower)**. It focuses on running legacy tools on modern **macOS (Apple Silicon)** and includes advanced hardware control instructions (fan speed, sensors).

**Supported Models:**
- AirPort Extreme 802.11ac (6th Gen) - **A1521**
- AirPort Time Capsule 802.11ac (5th Gen) - **A1470**

## 📂 Content

1.  **[macOS Apple Silicon Setup Guide](MACOS_SILICON_GUIDE.md)**
    *   **Host Setup:**
        *   [Running Python 2.7 on M1/M2/M3 chips](MACOS_SILICON_GUIDE.md#part-1-installing-python-27-on-apple-silicon).
        *   [Fixing SSH connection issues (RSA/DSS keys)](MACOS_SILICON_GUIDE.md#part-4-ssh-connection-fixes).
    
    *   **Device Modifications:**
        *   [Enabling SSH Access (Root Control)](MACOS_SILICON_GUIDE.md#part-3-enabling-ssh-access).
        *   [Region Unlocking: Removing 802.11ac limits & boosting power](MACOS_SILICON_GUIDE.md#part-5-region--power-unlock-speed-boost).
        *   [Advanced: Manual Fan Control & Sensor Monitoring](MACOS_SILICON_GUIDE.md#part-6-hardware-control--fan-modding-advanced).

2.  **[Software Backup](backup/)**
    *   A frozen copy of the original `airpyrt-tools` (Python) in case the original repo disappears.

## 🔗 Original Source
This guide is based on tools originally developed by [x56/airpyrt-tools](https://github.com/x56/airpyrt-tools).

## ⚠️ Disclaimer
Hardware modifications (fan control) are done at your own risk. Disabling the fan while using the original internal Power Supply Unit (PSU) leads to device failure.