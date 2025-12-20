# macOS Apple Silicon Setup & Hardware Control Guide

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


