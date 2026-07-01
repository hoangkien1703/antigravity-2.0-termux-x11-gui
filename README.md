# Antigravity 2.0 GUI on Termux-X11 (ARM64)

This repository provides a step-by-step guide and an automated patching tool to successfully install and run the official **Google Antigravity 2.0 Desktop GUI** inside a hardware-accelerated XFCE4 desktop environment on Android via **Termux-X11** and **Ubuntu PRoot**.

---

## 🛠️ The Problem: TCMalloc & 39-bit Virtual Address Spaces

Google's Antigravity desktop Electron app and its companion `language_server` binary are compiled using **TCMalloc** with hardcoded assumptions for a **48-bit** userspace virtual address (VA) space (common on standard Linux desktop distributions).

However, many Android devices and kernels limit the userspace virtual address layout to **39-bit**. When the binary attempts to allocate memory or use tag masks above the 39-bit limit, the Android kernel rejects the system call, causing a crash:

```text
MmapAligned() failed - unable to allocate with tag
TCMalloc assumes a 48-bit virtual address space size
FATAL ERROR: Out of memory trying to allocate internal tcmalloc data
```

### The Solution
This repository contains a pattern-matching script (`patch_binary_va39.py`) that scans the binary for TCMalloc tag operations, shifts, and memory alignment masks, lowering the address spaces and tags from bit 42/48 down to bit 35/39.

---

## 📋 Step-by-Step Installation Guide

Follow these steps from the beginning to install the Antigravity 2.0 GUI.

### Step 1: Install Prerequisites in Termux
Ensure your Termux package repository is up to date, and install Python, `proot-distro`, `curl`, and other essential utilities:

```bash
pkg update
pkg install python proot proot-distro curl ca-certificates -y
```

### Step 2: Set Up the Linux Desktop Environment (XFCE4 + Termux-X11)
To run a GUI application like Antigravity, you need a functional X11 server and window manager. The community standard uses XFCE4 with the hardware-accelerated Termux-X11 companion app.

1. Install the desktop environment via your preferred setup script (e.g., [termux-linux-setup](https://github.com/orailnoor/termux-linux-setup)).
2. Install the Ubuntu container via `proot-distro`:
   ```bash
   proot-distro install ubuntu
   ```
3. Start your desktop environment inside the Termux-X11 companion app:
   ```bash
   ~/start-linux.sh
   ```

---

### Step 3: Install the Antigravity 2.0 GUI Bundle in PRoot

1. Download the Linux ARM64 version of Google Antigravity 2.0 (`Antigravity.tar.gz`).
2. Extract the bundle into `/opt/antigravity/` inside your Ubuntu PRoot container rootfs (located on the host at `/data/data/com.termux/files/usr/var/lib/proot-distro/containers/ubuntu/rootfs/opt/antigravity/`).
3. Ensure the main executables are placed in:
   - **/opt/antigravity/Antigravity-arm64/antigravity** (main Electron executable)
   - **/opt/antigravity/Antigravity-arm64/resources/bin/language_server** (companion LSP server)

---

### Step 4: Patch the Binaries for 39-bit VA Compatibility
You must run the patch script on the **language_server** companion binary (and optionally any other auxiliary binaries like `webm_encoder` or `chrome_crashpad_handler`). 

> [!WARNING]
> Do **NOT** run the patch on the main `antigravity` Electron browser executable. Chromium uses PartitionAlloc (not TCMalloc), and modifying its instructions will corrupt the V8 Javascript engine, causing `Fatal error in v8::ToLocalChecked (Empty MaybeLocal)`.

Run the Python patcher on the host machine:

```bash
# Clone this repository
git clone https://github.com/hoangkien1703/antigravity-2.0-termux-x11-gui.git
cd antigravity-2.0-termux-x11-gui

# Run the patcher against the language_server binary in the Ubuntu rootfs
python3 patch_binary_va39.py /data/data/com.termux/files/usr/var/lib/proot-distro/containers/ubuntu/rootfs/opt/antigravity/Antigravity-arm64/resources/bin/language_server
```

You should see output indicating successful rewrites (e.g. `ubfx patches : 15`, `Tag constants rewritten`, etc.).

---

### Step 5: Configure the Launcher and Desktop Shortcut

1. Create a launcher script at `/data/data/com.termux/files/usr/bin/antigravity` on your Termux host:
   ```bash
   #!/data/data/com.termux/files/usr/bin/bash
   export DISPLAY=:0
   export PULSE_SERVER=127.0.0.1
   exec proot-distro login ubuntu --shared-tmp -- env DISPLAY=:0 PULSE_SERVER=127.0.0.1 /usr/local/bin/antigravity --no-sandbox "$@"
   ```
2. Make the launcher executable:
   ```bash
   chmod +x /data/data/com.termux/files/usr/bin/antigravity
   ```
3. Set up the desktop shortcut (`~/Desktop/antigravity.desktop`):
   ```desktop
   [Desktop Entry]
   Version=1.0
   Type=Application
   Name=Antigravity
   Comment=Google Antigravity 2.0 GUI
   Exec=/data/data/com.termux/files/usr/bin/antigravity %U
   Icon=/data/data/com.termux/files/home/.local/share/pixmaps/antigravity.png
   Terminal=false
   Categories=Development;IDE;
   StartupNotify=true
   StartupWMClass=Antigravity
   ```

---

## 🚀 Running Antigravity 2.0

Start the X11 desktop environment, launch the Termux-X11 companion app on your Android device, and double-click the **Antigravity** icon on the desktop (or run `antigravity` in your terminal).

To check logs:
- **Electron Logs:** `/root/.config/Antigravity/logs/main.log`
- **Language Server Logs:** `/root/.config/Antigravity/logs/language_server.log`

---

## 🤝 Acknowledgments
- Shouts to **@hjotha** for the core disassembly patching analysis.
- Community developers for packaging Termux-X11 / PRoot-distro wrappers.
