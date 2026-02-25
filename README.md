# Bliss Saver on Raspberry Pi - Complete Build Guide
## Running Classic Mac OS 9 Screensavers via SheepShaver Emulation

**Tested on:** Raspberry Pi 5 (4GB), Raspberry Pi OS Lite (64-bit)  
**Result:** Smooth, stable 640x480 screensaver display suitable for YouTube recording or home theater display

---

## Overview

This guide shows how to build a dedicated Mac OS 9 screensaver appliance using:
- **Raspberry Pi OS Lite** (minimal, no desktop environment)
- **SheepShaver** (PowerPC Mac emulator)
- **Mac OS 9.0.4** (classic Mac operating system)
- **Bliss Saver screensavers** (vintage abstract visualizations)

**Notes:**
- PowerPC FPU emulation works correctly on ARM64 (unlike 68k emulation)
- Boots directly to Mac OS 9 with no Linux visible

---

## Hardware Requirements

### Minimum
- Raspberry Pi 4B or newer
- 4GB+ microSD card
- HDMI monitor/TV
- 5V 3A Power supply
- USB keyboard
- Ethernet or WiFi connection (for setup)

---

## Software Requirements

### Files You'll Need

**Before starting, gather these files:**

1. **PowerPC ROM** (~4MB)
   - Old World ROM from Power Mac 7300/8600/9600
   - Example: `960FC647.ROM` or `PM9600.ROM`
   - Source: Macintosh Garden, Macintosh Repository

2. **Mac OS 9.0.4 Installer** (~600MB)
   - Bootable .toast or .iso format
   - Example: `MacOS-9-0-4.toast`
   - Source: Macintosh Garden

3. **Bliss Saver Screensavers**
   - PowerPC versions (NOT 68k versions)
   - Bliss Saver, Waves of Bliss, Space Garden
	 - Stuffit Expander (vintage mac software), to open .site files

**Get these on a USB drive before starting.**

---

## Installation Steps

### Part 1: Base System Setup

#### 1. Install Raspberry Pi OS Lite

```bash
# Flash Raspberry Pi OS Lite (64-bit) to SD card using Raspberry Pi Imager
# Enable SSH during setup if you want remote access
# Boot the Pi and login (default: pi/raspberry or set during setup)
```

#### 2. Update System

```bash
sudo apt update
sudo apt upgrade -y
```

#### 3. Install Build Dependencies

```bash
sudo apt install --no-install-recommends \
  git build-essential autoconf automake \
  libsdl2-dev libgtk2.0-dev libgmp-dev libmpfr-dev \
  xserver-xorg xinit x11-xserver-utils
```

**What these packages do:**
- `git` - Download source code
- `build-essential` - Compiler toolchain
- `autoconf`/`automake` - Build system
- `libsdl2-dev` - Graphics and input
- `libgtk2-dev` - GUI toolkit
- `libgmp-dev`/`libmpfr-dev` - Math libraries (required for ARM64)
- `xserver-xorg`/`xinit` - Minimal X11 server (no desktop environment)

---

### Part 2: Build SheepShaver

#### 4. Clone the ARM-Optimized Fork of SheepShaver

```bash
cd ~
git clone https://github.com/vaccinemedia/macemu
cd macemu/SheepShaver
```

#### 5. Create Symbolic Links

```bash
make links
```

**Required step** - creates symlinks between BasiliskII and SheepShaver shared code.

#### 6. Configure and Build

```bash
cd src/Unix
./autogen.sh --enable-addressing=direct,0x10000000
make -j4
```

**Build time:** 5-10 minutes on Pi 4/5

**The compiled binary will be at:**
```
~/macemu/SheepShaver/src/Unix/SheepShaver
```

#### 7. Verify Build

```bash
ls -lh ~/macemu/SheepShaver/src/Unix/SheepShaver
file ~/macemu/SheepShaver/src/Unix/SheepShaver
```

**Should show:** `ELF 64-bit LSB executable, ARM aarch64`

---

### Part 3: Prepare Mac OS 9 Files

#### 8. Create Directory Structure

```bash
mkdir -p ~/sheepshaver/Shared
cd ~/sheepshaver
```

#### 9. Transfer Files from USB Drive

**Insert USB drive and mount it:**

```bash
sudo mkdir -p /mnt/usb
sudo mount /dev/sda1 /mnt/usb

# Copy ROM file
cp /mnt/usb/*.ROM ~/sheepshaver/

# Copy Mac OS installer
cp /mnt/usb/*.toast ~/sheepshaver/

# Copy Bliss Saver files
cp -r /mnt/usb/BlissSaver* ~/sheepshaver/Shared/

# Unmount USB
cd ~
sudo umount /mnt/usb
```

**Verify files copied:**

```bash
ls -lh ~/sheepshaver/
```

**You should see:**
- ROM file (~4MB)
- Toast installer (~600MB)
- Shared folder with screensaver files

#### 10. Make Installer Read-Only

```bash
chmod 444 ~/sheepshaver/*.toast
```

#### 11. Create Blank Disk Image

```bash
cd ~/sheepshaver
dd if=/dev/zero of=macos9.img bs=1M count=800
```

**Creates an 800MB blank disk** - period-accurate size for Mac OS 9 + screensavers.

---

### Part 4: Configure SheepShaver

#### 12. Create Configuration File

```bash
mkdir -p ~/.config/SheepShaver
nano ~/.config/SheepShaver/prefs
```

**Paste this configuration (update ROM/installer filenames to match yours):**

```
# ==================== ROM & STORAGE ====================
rom /home/peter/sheepshaver/960FC647.ROM
disk /home/peter/sheepshaver/MacOS-9-0-4.toast
disk /home/peter/sheepshaver/macos9.img
extfs /home/peter/sheepshaver/Shared

# ==================== DISPLAY ====================
screen dga/640/480
windowmodes 0
screenmodes 0
frameskip 0
gfxaccel true
mag_rate 1.0
scale_nearest false
scale_integer false

# ==================== HARDWARE ====================
ramsize 268435456
modelid 14
cpu 4
keyboardtype 5

# ==================== PERIPHERALS ====================
seriala /dev/ttyS0
serialb /dev/ttyS1
nocdrom false
nosound true
nonet false

# ==================== INPUT ====================
keycodes false
mousewheelmode 1
mousewheellines 3
hardcursor true
hotkey 0
swap_opt_cmd true
init_grab false

# ==================== JIT ====================
jit false
jit68k false
cpuclock 0

# ==================== COMPATIBILITY ====================
ignoresegv true
ignoreillegal true
nogui true
noclipconversion false
idlewait true

# ==================== TIMING ====================
yearofs 0
dayofs 0

# ==================== AUDIO ====================
sound_buffer 0
name_encoding 0
dsp /dev/dsp
mixer /dev/mixer
```

**Key settings explained:**
- `screen dga/640/480` - Fullscreen at 640x480 (optimal for Bliss Saver)
- `frameskip 0` - No frame skipping for smooth animation
- `nosound true` - Sound disabled (prevents crashes, re-enable later if desired)
- `hardcursor true` - Fixes mouse/trackpad issues
- `jit false` - JIT is not available on ARM64

**Save and exit** (Ctrl+X, Y, Enter)

#### 13. Create Startup Script

```bash
nano ~/start-sheepshaver.sh
```

**Paste this:**

```bash
#!/bin/bash
xset s off -dpms s noblank
exec ~/macemu/SheepShaver/src/Unix/SheepShaver 2>/dev/null
```

**What this does:**
- Disables screen blanking
- Launches SheepShaver
- Suppresses debug messages

**Make executable:**

```bash
chmod +x ~/start-sheepshaver.sh
```

---

### Part 5: Install Mac OS 9

#### 14. First Boot - Installation

**From SSH (monitor must be connected to Pi):**

```bash
sudo xinit ~/start-sheepshaver.sh -- :0 vt1
```

**On your monitor you'll see:**

1. **Happy Mac icon**
2. **Mac OS 9 installer boots**
3. **Initialize dialog appears**
   - Click "Initialize"
   - Format as "Mac OS Extended"
   - Name it "Macintosh HD"
4. **Run installer**
   - Select Macintosh HD as destination
   - Choose "Custom Install"
   - **Uncheck:** Java, OpenDoc, QuickDraw GX, QuickDraw 3D, extra languages
   - Click Install
5. **Wait 10-15 minutes**
6. **When complete:** Special → Shut Down

**Installation tips:**
- Use USB keyboard for this step
- Don't restart yet - shut down completely
- Installation may crash at end (normal - it's complete)

#### 15. Configure Boot from Installed System

**Kill any running instances:**

```bash
sudo pkill SheepShaver
sudo pkill X
```

**Edit prefs to boot from hard disk:**

```bash
nano ~/.config/SheepShaver/prefs
```

**Move hard disk first (or comment out installer):**

```
disk /home/peter/sheepshaver/macos9.img
# disk /home/peter/sheepshaver/MacOS-9-0-4.toast
```

**Save and exit.**

#### 16. Boot Installed System

```bash
sudo xinit ~/start-sheepshaver.sh -- :0 vt1
```

**Should boot to Mac OS 9 desktop!**

**If it crashes at Finder preferences:**
- Already fixed with `nosound true` in prefs
- If still crashes, increase RAM to 512MB: `ramsize 536870912`

---

### Part 6: Install Bliss Saver

#### 17. Access Shared Files

**In Mac OS 9:**
- "Unix" volume appears on desktop
- Contains files from `~/sheepshaver/Shared/`

**Copy screensaver files to Mac HD:**
- If in .sit archives, extract with StuffIt Expander first
- Install screensavers per their instructions

#### 18. Configure Screensaver

**Method 1: Control Panels**
- Apple Menu → Control Panels → Appearance (or Desktop Pictures)
- Go to "Screen Saver" tab
- Select Bliss Saver
- Set activation time

**Method 2: Auto-Launch on Startup**
- Drag Bliss Saver app to System Folder → Startup Items
- Next boot, launches automatically

**Method 3: Run Manually**
- Double-click Bliss Saver app
- Enter fullscreen mode

---

### Part 7: Kiosk Mode Setup

#### 19. Enable Auto-Login

**Edit getty service:**

```bash
sudo systemctl edit --full getty@tty1
```

**Find the ExecStart line and change to:**

```
ExecStart=-/sbin/agetty --autologin peter --noclear %I ${TERM}
```

**Save and exit.**

**Reload systemd:**

```bash
sudo systemctl daemon-reload
```

#### 20. Auto-Start X11 on Boot

```bash
nano ~/.bash_profile
```

**Add this:**

```bash
if [ -z "$DISPLAY" ] && [ "$(tty)" = "/dev/tty1" ]; then
    startx ~/start-sheepshaver.sh
fi
```

**Save and exit.**

#### 21. Optional: Quiet Boot

**Suppress boot messages for cleaner startup:**

```bash
sudo nano /boot/firmware/cmdline.txt
```

**Add to end of the single line:**

```
quiet splash loglevel=0 logo.nologo vt.global_cursor_default=0
```

**Disable splash screen:**

```bash
sudo nano /boot/firmware/config.txt
```

**Add:**

```
disable_splash=1
avoid_warnings=1
```

**Save both files.**

#### 22. Test Auto-Start

```bash
sudo reboot
```

**After reboot:**
- Pi boots (~30 seconds)
- Auto-login occurs
- X11 starts automatically
- SheepShaver launches
- Mac OS 9 appears
- Bliss Saver runs

**No SSH, keyboard, or mouse needed!**

---

## Maintenance and Troubleshooting

### Remote Access

**SSH in for maintenance:**

```bash
ssh peter@raspberrypi
```

**Kill and restart SheepShaver:**

```bash
sudo pkill SheepShaver
sudo pkill X
sleep 2
sudo reboot
```

### Auto-Restart for Memory Leak

**Bliss Saver crashes after ~6 hours due to Mac OS 9 memory leak.**

**Solution: Reboot every 5 hours:**

```bash
crontab -e
```

**Add:**

```
0 */5 * * * sudo reboot
```

### Common Issues

**Problem:** X server exits immediately

**Solution:** Check prefs file paths match actual filenames exactly

---

**Problem:** "Cannot open display" error

**Solution:** Must use `xinit` or `startx`, not run SheepShaver directly

---

**Problem:** Mouse cursor invisible

**Solution:** Set `hardcursor true` in prefs

---

**Problem:** Installer says "copied to another drive"

**Solution:** `chmod 444` on toast file to make it read-only

---

**Problem:** Crashes at Finder preferences

**Solution:** Set `nosound true` in prefs

---

**Problem:** Jaggy/blocky animations

**Solution:** 
- Use PowerPC versions (not 68k)
- Reduce resolution to 640x480
- Set frameskip to 0

---

**Problem:** Can't get past Mac OS install dialog without keyboard

**Solution:** Connect USB keyboard temporarily for installation only

---

### File Locations Reference

```
~/macemu/SheepShaver/src/Unix/SheepShaver  - Compiled binary
~/.config/SheepShaver/prefs                - Configuration file
~/sheepshaver/960FC647.ROM                 - PowerPC ROM
~/sheepshaver/MacOS-9-0-4.toast            - Installer (read-only)
~/sheepshaver/macos9.img                   - Mac OS 9 hard disk
~/sheepshaver/Shared/                      - File transfer folder
~/start-sheepshaver.sh                     - Startup script
~/.bash_profile                            - Auto-start configuration
```

### Log Locations

```
/var/log/Xorg.0.log                        - X server log
sudo journalctl -b                         - Boot log
dmesg                                      - Kernel messages
```

---

## Performance Tips

### RAM Allocation

**Pi 3B (1GB total):**
```
ramsize 268435456       # 256MB
```

**Pi 4 (2-8GB total):**
```
ramsize 536870912       # 512MB
```

**Pi 5 (4-8GB total):**
```
ramsize 1073741824      # 1GB (maximum SheepShaver supports)
```

### Frame Skipping

**For smooth screensavers:**
```
frameskip 0             # No skipping
```

**For better CPU efficiency:**
```
frameskip 2             # Skip every 2nd frame (still looks good)
```

---

## Credits and Resources

**SheepShaver** - Christian Bauer and contributors  
**vaccinemedia fork** - Raspberry Pi optimizations  
**Bliss Saver** - Greg Jalbert (original developer)

**Resources:**
- SheepShaver: https://sheepshaver.cebix.net/
- vaccinemedia GitHub: https://github.com/vaccinemedia/macemu
- Emaculation Forums: https://www.emaculation.com/
- Macintosh Garden: https://macintoshgarden.org/
- Macintosh Repository: https://www.macintoshrepository.org/
- Bliss Saver: https://imaja.com 

---
