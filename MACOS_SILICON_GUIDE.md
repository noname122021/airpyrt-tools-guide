# macOS Apple Silicon Setup & Hardware Control Guide
## Complete AirPort Extreme Jailbreak Tutorial (Root SSH Access & Advanced Modifications)

This guide is intended for users running **macOS 12.3 Monterey (or newer)** on **Apple Silicon (M1/M2/M3)** chips. Tested on **macOS Sonoma 14.3**.

### ⚠️ For macOS Sonoma 14.4 (and newer)
Apple has removed the `airport` utility from its original location at `/System/Library/PrivateFrameworks/Apple80211.framework/Resources/airport`.

To restore functionality, copy the legacy binary provided in this repository to your system path:

```bash
sudo cp bin/airport /usr/local/bin/airport
sudo chmod +x /usr/local/bin/airport
```

It covers:
1.  Installing the legacy Python 2.7 environment on modern macOS.
2.  Fixing SSH connection issues with the AirPort Extreme.
3.  **Bonus:** Controlling the internal fan and monitoring sensors (NetBSD hacking).

---

## Prerequisites
Clone this repository to get the necessary airpyrt-tools:

```bash
git clone https://github.com/x56/airpyrt-tools
cd airport-tools
```

## Part 1: Installing Python 2.7 on Apple Silicon

macOS has removed Python 2.7, and standard installation methods fail on Apple Silicon because legacy Python cannot find system libraries (zlib, bzip2) in their new Homebrew locations.

### 1. Prerequisites
Install Homebrew and the required dependencies:

```bash
brew install pyenv zlib bzip2
```

### 2. Configure Shell
Ensure `pyenv` is initialized in your shell (Zsh is default on macOS). Run these commands once:

```bash
echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.zshrc
echo '[[ -d $PYENV_ROOT/bin ]] && export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.zshrc
echo 'eval "$(pyenv init -)"' >> ~/.zshrc
source ~/.zshrc
```

### 3. Build Python 2.7.18
Use specific compiler flags to point to Homebrew's library paths:

```bash
LDFLAGS="-L$(brew --prefix zlib)/lib -L$(brew --prefix bzip2)/lib"
CPPFLAGS="-I$(brew --prefix zlib)/include -I$(brew --prefix bzip2)/include"
pyenv install 2.7.18
```


### 4. Activate Environment
Navigate to this repository folder and activate the version:

```bash
pyenv local 2.7.18
```

---

## Part 2: Installing airpyrt-tools

### 1. Install Dependencies
The original tools use `pycrypto`, which is obsolete and difficult to build. Install the modern drop-in replacement `pycryptodome` instead:

```bash
pip install pycryptodome
```

### 2. Run Installation
Even with Python installed, `setup.py` may fail with linker errors (`ld: warning: search path not found`) because `setuptools` also needs to know where the libraries are.

**Tip:** If `setup.py` fails complaining about `pycrypto`, edit the `setup.py` file and change `install_requires` to:
`install_requires=["pycryptodome"]`

Run the installation with these exported flags:

```bash
export LDFLAGS="-L$(brew --prefix zlib)/lib -L$(brew --prefix bzip2)/lib"
export CFLAGS="-I$(brew --prefix zlib)/include -I$(brew --prefix bzip2)/include"
export CPPFLAGS="-I$(brew --prefix zlib)/include -I$(brew --prefix bzip2)/include"

python setup.py install
```

---

## Part 3: Enabling SSH Access

Before you can connect via SSH, you must enable the `dbug` flag on the AirPort Extreme using the `acp` tool (installed in Part 2).

**1. Enable SSH (dbug 0x3000):**
Replace `<IP_ADDRESS>` with your router's IP (usually `10.0.1.1`) and `<PASSWORD>` with your device password.

```bash
python -m acp -t <IP_ADDRESS> -p <PASSWORD> --setprop dbug 0x3000
python -m acp -t <IP_ADDRESS> -p <PASSWORD> --reboot
```

**2. Disable SSH (Optional):**
To secure the device later, you can disable the flag:

```bash
python -m acp -t <IP_ADDRESS> -p <PASSWORD> --setprop dbug 0x0000
python -m acp -t <IP_ADDRESS> -p <PASSWORD> --reboot
```

---

## Part 4: SSH Connection Fixes

The AirPort Extreme runs an old Dropbear SSH server that uses `ssh-rsa` and `ssh-dss` algorithms. Modern OpenSSH clients (included in macOS) disable these by default.

### Error Message
`Unable to negotiate with 10.0.1.1 port 22: no matching host key type found. Their offer: ssh-rsa,ssh-dss`

### The Fix
To connect purely via CLI:

```bash
ssh -oHostKeyAlgorithms=+ssh-rsa,ssh-dss -oPubkeyAcceptedKeyTypes=+ssh-rsa,ssh-dss root@10.0.1.1
```

### Permanent Fix
Add this to your `~/.ssh/config` file to avoid typing flags every time:

``` bash
Host 10.0.1.1
HostKeyAlgorithms +ssh-rsa,ssh-dss
PubkeyAcceptedKeyTypes +ssh-rsa,ssh-dss
```
*(Replace 10.0.1.1 with your router's IP)*

---

## Part 5: Region & Power Unlock (Speed Boost)

By connecting via SSH, you can remove regional restrictions (e.g., beamforming limitations or channel width caps in certain EU/Asian regions) and unlock higher power limits (FCC mode). This can significantly increase speed (e.g., from 150 Mbps to 800+ Mbps) by enabling previously locked 802.11ac capabilities.

**1. Connect via SSH:**
(See Part 4 for setup if you haven't already).

```bash
ssh root@10.0.1.1
```

**2. Run Unlock Commands:**
Once inside the SSH session, execute the generic `acp` tool to set the region to "0" (Universal) and SKU to "FCC" (US High Power).

```bash
acp -q syRe=0x00000000
acp static apple-sku=FCC
```

**3. Reboot:**
Exit SSH and reboot the router using the python tool (or power cycle it).

```bash
python -m acp -t 10.0.1.1 -p <PASSWORD> --reboot
```

---

## Part 6: Hardware Control & Fan Modding (Advanced)

> **⚠️ WARNING:** The following commands modify the low-level thermal management of the device.
> If you are using the original internal Power Supply Unit (PSU), **DO NOT stop the fan**. The PSU generates significant heat and will fail without active cooling.
> **Passive cooling is only recommended if you have modified the device to run on an external DC power source.**

Once logged in via SSH, you are in a restricted NetBSD 6.0 environment.

### 1. Monitoring Sensors
To view CPU temperature, Wi-Fi radio temperatures, and fan speed:

```bash
envstat
```

**Key metrics:**
*   `Fan_rpm`: Speed in Revolutions Per Minute.
*   `Fan_dcy`: Duty Cycle (PWM power sent to fan, 0-255).
*   `T_internal`: Main Logic Board / CPU temperature.

### 2. Manual Fan Control
Standard hardware controls (`hw.applfan`) are locked (`securelevel=1`). However, the **Closed Loop Thermal Management (CLTM)** debug override is writable.

Variable: `kern.cltm.override_fan`
Range: `0` to `255` (8-bit PWM).

#### Commands:

**Stop the Fan (Passive / Silent Mode):**
Values around 10 or lower provide insufficient voltage to spin the motor.

```bash
sysctl -w kern.cltm.override_fan=10
```

**Set Low Speed (Whisper Mode):**
Find a value that spins the fan at ~800 RPM.

```bash
sysctl -w kern.cltm.override_fan=128
```

**Set Max Speed:**

```bash
sysctl -w kern.cltm.override_fan=255
```

**Return to Automatic Control (Default):**

```bash
sysctl -w kern.cltm.override_fan=-1
```

### 3. Persistence
The AirPort filesystem is read-only. If the device reboots or loses power, the fan control resets to **Auto** (`-1`). You must re-apply the SSH command after every reboot.

---

## Part 7: ACP Properties (Advanced Configuration)

The AirPort Control Protocol (ACP) exposes hundreds of configuration properties that can be read and modified using 4-character codes. This allows deep customization beyond what AirPort Utility provides.

### 1. Using the `acp` Command

Once SSH is enabled and you're connected to the device, you can use the built-in `acp` tool:

**Get a property value:**
```bash
acp -q {property}
```

**Set a property value:**
```bash
acp -q {property}={value}
```

**List available properties:**
```bash
acp acpprop
```

### 2. Common Useful Properties

#### General Device Information
```bash
# Get device name
acp -q syNm

# Set device name
acp -q syNm=AirPort\ Extreme

# Get serial number
acp -q sySN

# Get firmware version
acp -q syVs

# Get MAC addresses
acp -q raMA  # Radio (Wi-Fi) MAC
acp -q laMA  # LAN MAC
acp -q waMA  # WAN MAC
```

#### Time & Timezone
```bash
# Get current time (Unix timestamp)
acp -q time

# Get timezone info
acp -q timz
```

#### Wi-Fi Configuration
Most Wi-Fi settings are nested in the `WiFi` property, but some shortcuts exist:
```bash
# Radio name (SSID) - via WiFi property
# Radio channel - via WiFi property
# Radio power - via WiFi property
```

#### Network Settings
```bash
# WAN IP address
acp -q waIP

# WAN subnet mask
acp -q waSM

# WAN router address (gateway)
acp -q waRA

# LAN IP address
acp -q laIP

# LAN subnet mask
acp -q laSM

# DHCP range begin
acp -q dhBg

# DHCP range end
acp -q dhEn
```

### 3. Important Notes

⚠️ **Always reboot after changing properties:**
```bash
# From SSH session on device
reboot

# Or from your Mac
python -m acp -t 10.0.1.1 -p <PASSWORD> --reboot
```

⚠️ **Some properties are read-only** and cannot be modified.

⚠️ **Invalid values can brick your device** or require a factory reset.

### 4. Full Documentation

For a complete reference of all ACP properties, codes, and their meanings, see:
- **[ACP Properties Wiki](https://github.com/samuelthomas2774/airport/wiki/ACP-properties)** - Comprehensive list
- **[acp command Wiki](https://github.com/samuelthomas2774/airport/wiki/acp-command)** - Command usage examples

---

## Part 8: USB Startup Scripts (Automation)

The AirPort Extreme can automatically execute custom scripts from a USB drive on startup. This enables advanced automation like applying custom firewall rules, configurations, or running services.

> ⚠️ **Security Warning**: This creates a potential security risk. Anyone with physical access and a USB drive could run malicious code. Use the hash verification system (see below) to protect against unauthorized scripts.

### 1. How It Works

The system works by:
1. Replacing `/etc/rc.local` with a custom script that mounts the USB drive (`/dev/dk0` → `/Volumes/dk0`)
2. Running `/Volumes/dk0/AirPort-startup.sh` if it exists and passes verification
3. Unmounting the drive and restarting `diskd` to restore normal USB functionality

### 2. Installation

**Prerequisites:**
- SSH access enabled (see Part 3)
- USB drive connected to the AirPort (formatted as HFS+, APFS, or FAT32)
- Clone the [airport repository](https://github.com/samuelthomas2774/airport) to your USB drive:
  ```bash
  # On your Mac, with USB drive mounted
  cd /Volumes/YOUR_USB_DRIVE
  git clone https://github.com/samuelthomas2774/airport.git
  ```

**Step 1: Connect via SSH**
```bash
ssh root@10.0.1.1
```

**Step 2: Verify Symbolic Links**
Check that the startup script hooks exist:
```bash
ls -la /etc | grep rc.local
```

You should see:
```
lrwxr-xr-x  1 root  wheel  19 Aug 30  2016 rc.local -> /mnt/Flash/rc.local
```

**Step 3: Mount USB Drive**
The USB is auto-mounted when accessed via SMB/AFP, or manually:
```bash
mount /dev/dk0 /Volumes/dk0
```

**Step 4: Copy Startup Scripts**
Assuming the `airport` repository is at the root of your USB drive (`/Volumes/dk0/airport/`):

```bash
cp /Volumes/dk0/airport/startup-scripts/AirPort-run.sh /mnt/Flash/rc.local
cp /Volumes/dk0/airport/startup-scripts/AirPort-rc.local.sh /mnt/Flash/AirPort-rc.local.sh
cp /Volumes/dk0/airport/startup-scripts/AirPort-run-on-ifup.sh /mnt/Flash/AirPort-run-on-ifup.sh
```

**Step 5: Set Up Security (Recommended)**
Generate a hash key to prevent unauthorized USB drives from running scripts:

```bash
cd /Volumes/dk0/airport
./startup-scripts/setup-security.sh
```

This creates:
- `/Volumes/dk0/AirPort-hash.key` - 1024 bytes of random data
- `/mnt/Flash/AirPort-hash.key` - SHA512 hash of that data

The startup script will verify the hash before running anything.

### 3. Creating Your Startup Script

Create `/AirPort-startup.sh` on your USB drive:

```bash
#!/bin/sh
# Example startup script

echo "[$(date)] AirPort startup script running..." > /AirPort-startup.log

# Apply custom firewall rules
/airport/pf/inject.sh /airport/pf/sed.txt

# Add more custom commands here

echo "[$(date)] Startup complete!" >> /AirPort-startup.log
```

Make it executable:
```bash
chmod +x /Volumes/dk0/AirPort-startup.sh
```

### 4. Removal

To remove startup scripts and return to default behavior:
```bash
ssh root@10.0.1.1
rm /mnt/Flash/rc.local
rm /mnt/Flash/AirPort-rc.local.sh
rm /mnt/Flash/AirPort-run-on-ifup.sh
reboot
```

### 5. Full Documentation

For detailed information, see:
- **[Startup Scripts Wiki](https://github.com/samuelthomas2774/airport/wiki/Startup-scripts)**
- **[Startup Scripts Source](https://github.com/samuelthomas2774/airport/tree/master/startup-scripts)**

---

## Part 9: Firewall Customization (Packet Filter)

The AirPort uses OpenBSD's `pf` (packet filter) for firewall and NAT. By default, the configuration is locked, but you can inject custom "anchors" (extension points) to add your own rules without modifying the main config.

### 1. What Are Anchors?

Anchors are placeholders in the pf config where you can load external rule files. The injection script adds these anchors:

- `AirPort-nat-before` - NAT rules before default NAT
- `AirPort-nat-after` - NAT rules after default NAT  
- `AirPort-filter-before` - Filter rules before default filters
- `AirPort-filter-after` - Filter rules after default filters
- `AirPort-natpmp-after` - Rules after NAT-PMP

### 2. Installation

**Prerequisites:**
- SSH access enabled
- USB drive with the cloned [airport repository](https://github.com/samuelthomas2774/airport)
- Startup scripts installed (Part 8) - *recommended but not required*

**Manual Injection (One-time):**

```bash
ssh root@10.0.1.1
mount /dev/dk0 /Volumes/dk0

# Run the injection script
/Volumes/dk0/airport/pf/inject.sh /Volumes/dk0/airport/pf/sed.txt
```

**Automatic Injection (via Startup Script):**

Add to your `/AirPort-startup.sh`:
```bash
#!/bin/sh
# Apply firewall modifications on every boot
/airport/pf/inject.sh /airport/pf/sed.txt

# Optional: run injection daemon (re-applies rules every minute)
# /airport/pf/injectd.sh > /AirPort-pf-injectd.sh.log 2>&1 &
```

### 3. Adding Custom Rules

Create rule files on your USB drive in `/AirPort-pf/`:

**Example: Port Forwarding `/AirPort-pf/nat-after.conf`**
```bash
# Forward port 8080 to internal server 10.0.1.100:80
rdr pass on $wan_if proto tcp from any to any port 8080 -> 10.0.1.100 port 80
```

**Example: Block Specific Traffic `/AirPort-pf/filter-after.conf`**
```bash
# Block outbound port 25 (SMTP)
block drop out quick on $wan_if proto tcp to any port 25

# Allow specific IP through firewall
pass in quick on $wan_if proto tcp from 203.0.113.42 to any
```

**Example: Custom NAT `/AirPort-pf/nat-before.conf`**
```bash
# Custom NAT rule
nat on $wan_if from 10.0.1.0/24 to any -> ($wan_if)
```

### 4. Applying Rules

After creating/modifying rule files, reload them:

```bash
ssh root@10.0.1.1

# Re-run injection script
/Volumes/dk0/airport/pf/inject.sh /Volumes/dk0/airport/pf/sed.txt
```

Or if using the daemon, it will auto-reload every minute.

### 5. Verification

Check if your rules are loaded:

```bash
# Show all pf rules
pfctl -s all

# Show NAT rules
pfctl -s nat

# Show filter rules
pfctl -s rules

# Show rules in a specific anchor
pfctl -a AirPort-filter-after -s rules
```

### 6. Important Notes

⚠️ **Rules are not persistent across reboots** - Use startup scripts (Part 8) to re-apply on boot

⚠️ **Invalid syntax can break networking** - Test rules carefully

⚠️ **Variables available:**
- `$wan_if` - WAN interface (usually `arge0`)
- `$lan_if` - LAN interface (usually `bridge0`)
- `$priv4` - Private IPv4 ranges table
- `$priv6` - Private IPv6 ranges table

### 7. Full Documentation

For detailed information:
- **[pf Modification README](https://github.com/samuelthomas2774/airport/tree/master/pf)**
- **[OpenBSD PF User's Guide](https://www.openbsd.org/faq/pf/)** - Official pf documentation

---

## Appendix: Additional Resources

### Official Documentation
- **[AirPort Utility](https://github.com/samuelthomas2774/airport/wiki/AirPort-Utility)** - Information about the official configuration tool
- **[AirPort Status Codes](https://github.com/samuelthomas2774/airport/wiki/AirPort-status-codes)** - Diagnostic codes and troubleshooting
- **[Static Routes](https://github.com/samuelthomas2774/airport/wiki/Static-routes)** - Configuring custom routing

### Community Resources
- **[samuelthomas2774/airport Wiki](https://github.com/samuelthomas2774/airport/wiki)** - Comprehensive hacking documentation
- **[x56/airpyrt-tools](https://github.com/x56/airpyrt-tools)** - Python implementation of ACP protocol

### Alternative Tools
- **[node-acp](https://github.com/samuelthomas2774/airport/wiki/node-acp)** - Node.js implementation with full encryption support and IPv6

---

**End of Guide**
