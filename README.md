# AirPort Extreme & Time Capsule 802.11ac Jailbreak Guide & Tools

This repository serves as a comprehensive documentation hub for **jailbreaking** and modifying **Apple AirPort Extreme & Time Capsule 802.11ac (Tower)** devices. It covers obtaining root SSH access, removing regional restrictions, hardware control (fan speed, sensor monitoring), and advanced customization. The guide focuses on running the Python 3 AirPyrt tools on modern **macOS (Apple Silicon)** systems.

> **What is AirPort Jailbreak?** Gaining root shell access to your AirPort device to unlock advanced features, remove regional Wi-Fi restrictions, customize firewall rules, and control hardware that Apple's official AirPort Utility doesn't expose.

**Supported Models (802.11ac "Tower" Generation):**
- **AirPort Extreme** (6th Generation) - **A1521**
- **AirPort Time Capsule** (5th Generation / AC) - **A1470** (2TB & 3TB models)

**Legacy Models (802.11n "Flat" Generation):**
- **AirPort Extreme** (5th Generation) - **A1408**
- **AirPort Time Capsule** (4th Generation) - **A1409**

*Note: While the 802.11ac Time Capsule is technically the 5th generation of its specific line, it is part of the 6th generation of AirPort base stations (the "AC" wave). Legacy models (A1408/A1409) support SSH access but may have different hardware control parameters.*

## 📂 Content

1.  **[macOS Apple Silicon Setup Guide](MACOS_SILICON_GUIDE.md)**
    *   **Host Setup:**
        *   [Installing Python 3 AirPyrt tools on M1/M2/M3 chips](MACOS_SILICON_GUIDE.md#part-1-installing-python-3-on-apple-silicon).
        *   [Fixing SSH connection issues (RSA/DSS keys)](MACOS_SILICON_GUIDE.md#part-4-ssh-connection-fixes).
    
    *   **Device Modifications:**
        *   [Enabling SSH Access (Root Control)](MACOS_SILICON_GUIDE.md#part-3-enabling-ssh-access).
        *   [Region Unlocking: Removing 802.11ac limits & boosting power](MACOS_SILICON_GUIDE.md#part-5-region--power-unlock-speed-boost).
        *   [Advanced: Manual Fan Control & Sensor Monitoring](MACOS_SILICON_GUIDE.md#part-6-hardware-control--fan-modding-advanced).
        *   [Time Capsule: HDD Spindown (Sleep) Settings](HDD_SPINDOWN_GUIDE.md) (Experimental/Untested).
        *   [ACP Properties Reference (Advanced Configuration)](MACOS_SILICON_GUIDE.md#part-7-acp-properties-advanced-configuration).
        *   [USB Startup Scripts (Automation)](MACOS_SILICON_GUIDE.md#part-8-usb-startup-scripts-automation).
        *   [Firewall Customization (Packet Filter)](MACOS_SILICON_GUIDE.md#part-9-firewall-customization-packet-filter).

2.  **[Software Backup](backup/)**
    *   **`airpyrt-tools-main.zip`** - Frozen copy of [jamesyc/airpyrt-tools](https://github.com/jamesyc/airpyrt-tools) at `ad904eb5d851c308cbbe1e04f8e90ccbe6efb355` (Python 3 ACP implementation)
    *   **`airport-main.zip`** - Frozen copy of [samuelthomas2774/airport](https://github.com/samuelthomas2774/airport) (startup scripts, firewall mods, wiki resources)
    *   These archives are preserved in case the original repositories become unavailable.

## 🔗 Sources
- **Python 3 Tools**: [jamesyc/airpyrt-tools](https://github.com/jamesyc/airpyrt-tools) - current ACP protocol implementation and device management
- **Original Python Tools**: [x56/airpyrt-tools](https://github.com/x56/airpyrt-tools) - original ACP implementation that the Python 3 fork updates
- **Advanced Hacking**: [samuelthomas2774/airport](https://github.com/samuelthomas2774/airport) - Startup scripts, firewall mods, and comprehensive ACP properties documentation ([Wiki](https://github.com/samuelthomas2774/airport/wiki))

## ⚠️ Disclaimer
Hardware modifications (fan control, disk sleep) are done at your own risk. Disabling the fan while using the original internal Power Supply Unit (PSU) leads to device failure. HDD spindown settings are experimental and **untested** (see [Issue #1](https://github.com/noname122021/airpyrt-tools-guide/issues/1)).

---

### 🔍 Keywords
Airport Extreme jailbreak, Airport Time Capsule root access, A1521 SSH, A1470 firmware modification, Airport Extreme hacking, ACP protocol, region unlock, fan control, Apple Silicon Python 3, macOS Sonoma airport utility, Airport firewall customization, startup scripts, packet filter
