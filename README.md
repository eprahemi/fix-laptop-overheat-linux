# Modern Linux Laptops Overheating Cooldown Guide

> **Stop your hybrid-graphics laptop from overheating — on any distro.**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Linux](https://img.shields.io/badge/Linux-Compatible-FCC624?logo=linux&logoColor=black)](https://kernel.org)
[![NVIDIA](https://img.shields.io/badge/NVIDIA-Optimus-76B900?logo=nvidia&logoColor=white)](https://www.nvidia.com)
[![Intel](https://img.shields.io/badge/Intel-VAAPI-0071C5?logo=intel&logoColor=white)](https://www.intel.com)

---

## 📸 The Problem

Your laptop has **two graphics cards**:

| GPU | Type | Typical Temp (Idle) |
|-----|------|-------------------|
| 🟢 Integrated (Intel/AMD) | Efficient | **48–55°C** |
| 🔴 NVIDIA Dedicated | Powerful, **hot** | **54–65°C** |

Both cards **share the same heat pipe and fan**. Even when you're just browsing the web, the NVIDIA card stays partially awake, dumping heat into the system. That's why your laptop feels hot and the fan won't shut up.

**This guide fixes that** — on **Fedora**, **Ubuntu/Debian**, or **Arch Linux**.

---

## 📋 Table of Contents

- [What's Included](#-whats-included)
- [Before You Start](#-before-you-start)
- [Step 1 — Disable the NVIDIA GPU](#step-1--disable-the-nvidia-gpu)
- [Step 2 — Install TLP (CPU Power Management)](#step-2--install-tlp-cpu-power-management)
- [Step 3 — Hardware Video Acceleration (VAAPI)](#step-3--hardware-video-acceleration-vaapi)
- [Step 4 — Enable VAAPI in Chrome](#step-4--enable-vaapi-in-chrome)
- [Step 5 — Optimize Discord (Flatpak)](#step-5--optimize-discord-flatpak)
- [✅ Verify Everything](#-verify-everything)
- [🔄 Reverse — Get Your NVIDIA Back](#-reverse--get-your-nvidia-back)
- [🆘 Troubleshooting](#-troubleshooting)
- [📦 Quick Reference (Cheat Sheet)](#-quick-reference-cheat-sheet)

---

## 🚀 What's Included

| Step | What it does | Expected temp drop |
|------|-------------|-------------------|
| **1** — Disable NVIDIA | Kills power to the dedicated GPU entirely | **−5 to −10°C** |
| **2** — TLP | Forces CPU into powersave mode, deep-sleeps unused hardware | **−3 to −8°C** |
| **3** — VAAPI | Lets the Intel GPU handle video decoding instead of the CPU | **−2 to −5°C** |
| **4** — Chrome flag | Tells Chrome to use VAAPI for YouTube/Netflix | **−1 to −3°C** |
| **5** — Discord | Runs Discord natively on Wayland with GPU acceleration | **−1 to −2°C** |

**Total expected improvement: 10–20°C at idle.** 🧊

---

## 📝 Before You Start

- **Important First:** You need to update your linux and your Linux Distributions in the terminal.
- ├── Fedora
│   └── sudo dnf upgrade

├── Ubuntu / Debian
│   └── sudo apt update && sudo apt upgrade

└── Arch
    └── sudo pacman -Syu
- **You need:** A working internet connection and sudo (admin) access.
- **Tested on:** Fedora 44, MSI Thin GF63 12UCX (i5-12450H + RTX 2050).
- **Should work on:** Any Linux laptop with NVIDIA Optimus (hybrid graphics).
- **Will it break gaming?** Yes, on Linux. The NVIDIA GPU is disabled. See the [reverse section](#-reverse--get-your-nvidia-back) to bring it back when you need it.
- **Don't see your distro?** Translate the package manager:
  - `dnf` → Fedora
  - `apt` → Debian/Ubuntu/Mint/Pop!\_OS
  - `pacman` → Arch/Manjaro/EndeavourOS

---

## Step 1 — Disable the NVIDIA GPU

This is the single biggest improvement. We completely shut off power to the NVIDIA card.

### 1.1 Install EnvyControl

[EnvyControl](https://github.com/sunwire-pl/envycontrol) is a tool that switches between graphics modes safely.

<!-- FEDORA (DEFAULT) -->
<details open>
<summary><b>🐧 Fedora (default)</b></summary>

```bash
# Enable the EnvyControl repository
sudo dnf copr enable sunwire/envycontrol -y

# Install EnvyControl
sudo dnf install envycontrol -y
```
</details>

<details>
<summary><b>🐟 Ubuntu / Debian / Pop!_OS</b></summary>

```bash
# Add the EnvyControl PPA
sudo add-apt-repository ppa:ubuntuhandbook1/envycontrol -y

# Update package list and install
sudo apt update
sudo apt install envycontrol -y
```

> **Note for Pop!_OS:** You can also use the built-in system76-power tool instead — `sudo system76-power graphics integrated` — but EnvyControl works too.
</details>

<details>
<summary><b>📐 Arch Linux / Manjaro / EndeavourOS</b></summary>

```bash
# Install from AUR (use your preferred AUR helper)
yay -S envycontrol
# or
paru -S envycontrol
```

> If you don't have an AUR helper, install one first: `sudo pacman -S --needed git base-devel && git clone https://aur.archlinux.org/yay.git && cd yay && makepkg -si`
</details>

### 1.2 Switch to Integrated Mode

```bash
sudo envycontrol -s integrated
```

<details>
<summary>📖 What does this do?</summary>

This rebuilds the system boot files (initramfs) to blacklist the NVIDIA driver. After reboot, the NVIDIA card will receive no power and no driver.

**Expected output:**
```
Switching to integrated mode
Successfully disabled nvidia-persistenced.service
Rebuilding the initramfs...
Successfully rebuilt the initramfs!
Operation completed successfully
Please reboot your computer for changes to take effect!
```
</details>

### 1.3 Reboot

```bash
reboot
```

Wait for the laptop to restart and log back in.

### 1.4 Verify

```bash
nvidia-smi
```

**✅ Expected (good):**
```
NVIDIA-SMI has failed because it couldn't communicate with the NVIDIA driver.
```

This means the GPU is **completely off**. Don't worry about the error — we want it.

**❌ If you still see GPU stats:** Re-run `sudo envycontrol -s integrated` and reboot again.

---

## Step 2 — Install TLP (CPU Power Management)

Replace your default power manager with [TLP](https://linrunner.de/tlp/) — it gives much deeper control over CPU scaling, PCI Express sleep states, USB autosuspend, and more.

<!-- FEDORA (DEFAULT) -->
<details open>
<summary><b>🐧 Fedora (default)</b></summary>

```bash
# Remove the default power manager (conflicts with TLP)
sudo dnf remove tuned tuned-ppd -y

# Install TLP
sudo dnf install tlp tlp-rdw -y
```
</details>

<details>
<summary><b>🐟 Ubuntu / Debian / Pop!_OS</b></summary>

```bash
# Remove the default power manager (if installed)
sudo apt remove power-profiles-daemon -y

# Install TLP
sudo apt install tlp tlp-rdw -y
```
</details>

<details>
<summary><b>📐 Arch Linux / Manjaro / EndeavourOS</b></summary>

```bash
# Remove power-profiles-daemon (if installed)
sudo pacman -R power-profiles-daemon

# Install TLP
sudo pacman -S tlp tlp-rdw
```
</details>

### Enable TLP and switch to battery mode

```bash
# Enable TLP to start automatically on boot, and start it now
sudo systemctl enable tlp --now

# Switch to battery-saving profile
sudo tlp bat
```

**Verify:**
```bash
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
```

**Expected output:** `powersave`

This means the CPU runs at lower speeds when idle and only ramps up when you actually do something.

---

## Step 3 — Hardware Video Acceleration (VAAPI)

VAAPI (Video Acceleration API) lets your **Intel integrated GPU** handle video decoding instead of your CPU. The Intel GPU has dedicated hardware for H.264, H.265, VP9, and AV1 that uses **far less power**.

<!-- FEDORA (DEFAULT) -->
<details open>
<summary><b>🐧 Fedora (default)</b></summary>

```bash
sudo dnf install intel-media-driver libva-utils -y
```
</details>

<details>
<summary><b>🐟 Ubuntu / Debian / Pop!_OS</b></summary>

```bash
sudo apt install intel-media-driver libva-utils -y
```
</details>

<details>
<summary><b>📐 Arch Linux / Manjaro / EndeavourOS</b></summary>

```bash
sudo pacman -S intel-media-driver libva-utils
```
</details>

### Verify

```bash
vainfo
```

**Expected output (first lines):**
```
libva info: VA-API version 1.23.0
libva info: Trying to open /usr/lib64/dri-nonfree/iHD_drv_video.so
vainfo: Driver version: Intel iHD driver for Intel(R) Gen Graphics - 26.1.5
```

---

## Step 4 — Enable VAAPI in Chrome / Edge

Chromium-based browsers (Chrome, Edge, Brave, Opera) need a hidden flag flipped to use VAAPI.

### For Chrome / Edge / Brave

1. Open your browser
2. Paste this into the address bar:

   ```
   chrome://flags/#disable-accelerated-video-decode
   ```
   
   *(If you're on Edge, `edge://flags/#disable-accelerated-video-decode` also works)*

3. ⚠️ **Read carefully:** This flag says "Hardware-accelerated video decode" but the name starts with "disable."

   | If you set it to... | Actual result |
   |---|---|
   | **Enabled** | Video uses **CPU** ❌ |
   | **Disabled** | Video uses **GPU** ✅ |

4. Set it to **Disabled**, then click **Relaunch**.

After relaunch, YouTube, Netflix, and any video playback use the Intel GPU instead of your CPU.

<details>
<summary>🦊 Using Firefox instead?</summary>

1. Open Firefox → `about:config` → Accept the risk
2. Search for `media.ffmpeg.vaapi.enabled`
3. Toggle it to **true**
</details>

---

## Step 5 — Optimize Discord (Flatpak)

<details open>
<summary><b>Using Flatseal (GUI — recommended)</b></summary>

```bash
# Install Flatseal
# Fedora:
sudo dnf install flatseal -y
```

```bash
# Ubuntu/Debian:
sudo apt install flatseal -y
```
```bash
# Arch:
sudo pacman -S flatseal
```

1. Open **Flatseal**
2. Click **Discord** in the sidebar
3. Set these permissions:

| Setting | Value |
|---------|-------|
| 🖥️ **Inherit Wayland Socket** | ✅ ON |
| 🎮 **GPU Acceleration** | ✅ ON |
| ❌ **Fallback to X11** | ⛔ OFF |

4. Restart Discord
</details>

<details>
<summary><b>Terminal (alternative)</b></summary>

```bash
flatpak override --user com.discordapp.Discord \
  --socket=wayland \
  --device=dri \
  --env=DISABLE_GPU_ACCELERATED_VIDEO_DECODE=0
```
</details>

---

## ✅ Verify Everything

### Check temperatures

```bash
cat /sys/class/thermal/thermal_zone*/temp
```

**How to read the output:**
```
50000   ← 50.0°C  (CPU package — important one)
20000   ← 20.0°C  (ambient sensor, ignore)
50      ← ~0°C    (NVIDIA zone — no longer active)
46000   ← 46.0°C
48000   ← 48.0°C
48000   ← 48.0°C
48000   ← 48.0°C
```

**Good result:** CPU at **48–55°C** at idle. Before this guide, it was likely 60–70°C.

### Check CPU governor

```bash
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
```

Should say: `powersave`

### Check TLP status

```bash
sudo tlp-stat -s
```

Look for `TLP mode: BAT` near the top.

---

## 🔄 Reverse — Get Your NVIDIA Back

### Option A: Just re-enable NVIDIA (keep TLP)

```bash
sudo envycontrol -s nvidia
reboot
```

**Verify:** `nvidia-smi` — you should see GPU stats.

### Option B: Full factory reset

<!-- FEDORA -->
<details open>
<summary><b>🐧 Fedora</b></summary>

```bash
sudo envycontrol -s nvidia
sudo systemctl disable --now tlp
sudo dnf remove tlp tlp-rdw -y
sudo dnf install tuned tuned-ppd -y
sudo systemctl enable --now tuned
sudo dnf remove envycontrol -y
sudo dnf copr disable sunwire/envycontrol
reboot
```
</details>

<details>
<summary><b>🐟 Ubuntu / Debian / Pop!_OS</b></summary>

```bash
sudo envycontrol -s nvidia
sudo systemctl disable --now tlp
sudo apt remove tlp tlp-rdw -y
sudo apt install power-profiles-daemon -y
sudo systemctl enable --now power-profiles-daemon
sudo apt remove envycontrol -y
sudo add-apt-repository --remove ppa:ubuntuhandbook1/envycontrol
reboot
```
</details>

<details>
<summary><b>📐 Arch Linux / Manjaro / EndeavourOS</b></summary>

```bash
sudo envycontrol -s nvidia
sudo systemctl disable --now tlp
sudo pacman -R tlp tlp-rdw
sudo pacman -S power-profiles-daemon
sudo systemctl enable --now power-profiles-daemon
sudo pacman -R envycontrol
reboot
```
</details>

### Expected temperatures after reversing

| Scenario | Idle Temp | Fan Noise | Battery Life |
|----------|-----------|-----------|-------------|
| 🧊 **Integrated mode** (this guide) | **48–55°C** | Silent/quiet | ✅ Long |
| 🌡️ **NVIDIA enabled** | 55–65°C | Spins often | ⚠️ Shorter |
| 🔥 **Gaming with NVIDIA** | 70–85°C | Loud | ❌ Very short |

---

## 🆘 Troubleshooting

<details>
<summary><b>Commands say "command not found"</b></summary>

Make sure you typed the command exactly. If a package won't install, try updating first:

```bash
# Fedora
sudo dnf update
```

```bash
# Ubuntu/Debian
sudo apt update
```

```bash
# Arch
sudo pacman -Syu
```
</details>

<details>
<summary><b>Permission denied or not in sudoers</b></summary>

You need admin access. Ask the person who set up your laptop to add you:

```bash
sudo usermod -aG wheel YOUR_USERNAME   # Fedora
sudo usermod -aG sudo YOUR_USERNAME    # Ubuntu/Debian
```

Then log out and back in.
</details>

<details>
<summary><b>Temperatures didn't drop after Step 1</b></summary>

Wait 2–3 minutes after boot. If still high:

```bash
lspci | grep -i nvidia
```

If NVIDIA shows up, run `sudo envycontrol -s integrated` again and reboot.
</details>

<details>
<summary><b>Steam or games don't work</b></summary>

All graphics run on the Intel iGPU after this guide. For gaming, see the [Reverse section](#-reverse--get-your-nvidia-back) and use **Option A** or **C**.
</details>

<details>
<summary><b>vainfo shows errors</b></summary>

The Intel driver may not be installed correctly:

```bash
# Fedora
sudo dnf reinstall intel-media-driver -y
# Ubuntu/Debian
sudo apt install --reinstall intel-media-driver -y
# Arch
sudo pacman -S intel-media-driver
```

Then re-run `vainfo`.
</details>

<details>
<summary><b>I have an AMD CPU (not Intel)</b></summary>

VAAPI works on AMD too! Install the AMD driver instead:

```bash
# Fedora
sudo dnf install mesa-va-drivers -y
# Ubuntu/Debian
sudo apt install mesa-va-drivers -y
# Arch
sudo pacman -S mesa-va
```

The rest of the guide is the same.
</details>

---

## 📦 Quick Reference (Cheat Sheet)

### 🐧 Fedora

```bash
# STEP 1 — Disable NVIDIA
sudo dnf copr enable sunwire/envycontrol -y
sudo dnf install envycontrol -y
sudo envycontrol -s integrated
reboot

# STEP 2 — TLP
sudo dnf remove tuned tuned-ppd -y
sudo dnf install tlp tlp-rdw -y
sudo systemctl enable tlp --now
sudo tlp bat

# STEP 3 — VAAPI
sudo dnf install intel-media-driver -y
vainfo

# STEP 4 — Chrome
# chrome://flags/#enable-accelerated-video → Enable → Relaunch

# STEP 5 — Discord
# Flatseal → Discord → Wayland ON, GPU Accel ON, Fallback OFF

# VERIFY
cat /sys/class/thermal/thermal_zone*/temp
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
```

### 🐟 Ubuntu / Debian / Pop!_OS

```bash
# STEP 1 — Disable NVIDIA
sudo add-apt-repository ppa:ubuntuhandbook1/envycontrol -y
sudo apt update && sudo apt install envycontrol -y
sudo envycontrol -s integrated
reboot

# STEP 2 — TLP
sudo apt remove power-profiles-daemon -y
sudo apt install tlp tlp-rdw -y
sudo systemctl enable tlp --now
sudo tlp bat

# STEP 3 — VAAPI
sudo apt install intel-media-driver -y
vainfo

# STEP 4 — Chrome
# chrome://flags/#enable-accelerated-video → Enable → Relaunch

# VERIFY
cat /sys/class/thermal/thermal_zone*/temp
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
```

### 📐 Arch Linux / Manjaro / EndeavourOS

```bash
# STEP 1 — Disable NVIDIA
yay -S envycontrol       # or paru -S envycontrol
sudo envycontrol -s integrated
reboot

# STEP 2 — TLP
sudo pacman -R power-profiles-daemon
sudo pacman -S tlp tlp-rdw
sudo systemctl enable tlp --now
sudo tlp bat

# STEP 3 — VAAPI
sudo pacman -S intel-media-driver
vainfo

# STEP 4 — Chrome
# chrome://flags/#enable-accelerated-video → Enable → Relaunch

# VERIFY
cat /sys/class/thermal/thermal_zone*/temp
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
```

---

## 📚 Bonus Appendix

### Why does my laptop get hot?

Modern laptops with **NVIDIA Optimus** have two GPUs. The NVIDIA GPU is designed to turn off when idle, but in practice it often stays in a low-power "P8" state drawing 1–3W. On a thin laptop with a shared heat pipe, that's enough to keep temperatures 10°C higher than necessary.

### Why TLP?

The default power manager (`power-profiles-daemon` on Ubuntu/Arch, `tuned` on Fedora) only offers three simple profiles. TLP gives you granular control over CPU scaling governors, PCI Express ASPM, SATA link power, USB autosuspend, WiFi power saving, and more.

### What about AMD GPUs?

If you have an AMD dedicated GPU instead of NVIDIA, the process is different — AMD GPUs are better at entering deep sleep states. You can still use TLP and VAAPI from this guide.

### Does this affect Windows?

No. Windows is on a separate partition — this only affects Linux.

### Can I just use TLP without disabling NVIDIA?

Yes. Skip Step 1 and just do Steps 2–5. The cooling improvement will be smaller but still noticeable.

---

## 📄 License

This guide is provided under the **MIT License**. Use it, share it, fork it, translate it — just give credit.

---

## ⭐ Support

If this guide helped you, consider **starring the repo**! It helps others find it.

[![GitHub stars](https://img.shields.io/github/stars/eprahemi/fix-laptop-overheat-linux?style=social)](https://github.com/eprahemi/fix-laptop-overheat-linux)

---

*29 June 2026 · Created by [Eprahemi](https://github.com/eprahemi)*
