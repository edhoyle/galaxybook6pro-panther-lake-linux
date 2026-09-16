# Enabling camera and audio on the Samsung Galaxy Book6 Pro under Pop!_OS — clean install runbook

**Hardware:** Samsung Galaxy Book6 Pro (Panther Lake), model `NP940XJG-LG1BR`.
- Camera: Samsung/SmartSens SC200PC sensor, ACPI HID `SSLC2000`, behind an Intel IPU7.
- Audio: two Cirrus Logic CS35L57 speaker amplifiers over SoundWire/SDCA, plus a cs42l45 codec for headphones/mic.

**Camera root cause:** neither the kernel nor libcamera recognize this sensor natively. You need: (1) a libcamera recent enough for the `simple`/SoftISP pipeline, (2) an out-of-tree sensor driver and `ipu-bridge` patch (the `sc200pc-linux` project), and (3) a PipeWire SPA module rebuilt against that libcamera so normal apps (Chrome, Firefox) can see the camera.

**Audio root cause:** the `alsa-ucm-conf` package shipped by Pop!_OS (System76 build) is behind Ubuntu's own package and is missing the UCM2 profile for this board's `cs42l45` codec — without that file, WirePlumber can leave the speaker sink in an inconsistent state (muted / no active route) even though the hardware and SOF firmware load fine. See section 8.

**Pop!_OS version used:** Pop!_OS 24.04 LTS (COSMIC), Ubuntu `noble` base, kernel `7.1.5-76070105-generic`. Check yours with `cat /etc/os-release` or `pop-os-select`.

This guide assumes a clean Pop!_OS install. Follow the steps in order — each section depends on the previous one.

---

## 0. Before you start

- Confirm the kernel version: `uname -r`. Tested on `7.1.5-76070105-generic`. Very different kernel versions may require rebuilding the DKMS modules (they rebuild automatically if `dkms` is set up correctly).
- Everything below assumes a Wayland session (`echo $XDG_SESSION_TYPE` → `wayland`), the default on Pop!_OS with COSMIC.

---

## 1. Build libcamera 0.7.0 from source

Pop!_OS (Ubuntu/noble base) only ships `libcamera-ipa 0.2.0` in its repos — far too old for the `simple` pipeline this sensor needs. libcamera has to be built by hand.

```bash
sudo apt install -y git meson ninja-build pkg-config \
  libyaml-dev python3-yaml python3-ply python3-jinja2 \
  libgnutls28-dev openssl libtiff-dev libjpeg-dev \
  libevent-dev libdrm-dev libexif-dev \
  libgles2-mesa-dev libegl1-mesa-dev

git clone https://git.libcamera.org/libcamera/libcamera.git ~/libcamera
cd ~/libcamera
git checkout v0.7.0

meson setup build
meson configure build -Dpipelines=ipu3,simple,uvcvideo,vimc -Dipas=ipu3,simple
ninja -C build
sudo ninja -C build install
sudo ldconfig
```

Confirm it installed cleanly:

```bash
cam -l
```

It should list `Available cameras:` with no symbol errors (`symbol lookup error`). If that error shows up, see **Troubleshooting → libcamera version conflicts** below — it's the most common issue in this whole process.

**Note on libgles2-mesa-dev:** required because SoftISP uses GPU acceleration (EGL/GLES2) for debayering. To avoid that dependency, build with `-Dsoftisp-gpu-accel=disabled` instead, at the cost of higher CPU usage (~63% of one core vs. ~9% with GPU, per the reference project's measurements).

---

## 2. Install the sensor's DKMS modules (`sc200pc-linux` project)

The reference project (originally packaged for Arch Linux) lives at:
`https://github.com/Jabbslad/sc200pc-linux`

Since it's packaged for AUR/`makepkg`, on Pop!_OS you need to pull out the real contents and build the DKMS packages by hand.

```bash
sudo apt install dkms

git clone https://github.com/Jabbslad/sc200pc-linux ~/sc200pc-linux
cd ~/sc200pc-linux
```

### 2.1 — `ipu-bridge-sslc2000` module (kernel patch that recognizes the sensor)

```bash
sudo mkdir -p /usr/src/ipu-bridge-sslc2000-1.0
sudo cp packaging/ipu-bridge-sslc2000-dkms/{Kbuild,Makefile,ipu-bridge.c,dkms.conf} \
  /usr/src/ipu-bridge-sslc2000-1.0/
sudo sed -i 's/@PKGVER@/1.0/' /usr/src/ipu-bridge-sslc2000-1.0/dkms.conf

sudo dkms add -m ipu-bridge-sslc2000 -v 1.0
sudo dkms build -m ipu-bridge-sslc2000 -v 1.0
sudo dkms install -m ipu-bridge-sslc2000 -v 1.0
```

### 2.2 — `sc200pc` module (sensor's V4L2 driver)

```bash
sudo mkdir -p /usr/src/sc200pc-0.9.0
sudo cp packaging/sc200pc-dkms/{Kbuild,Makefile,sc200pc.c,dkms.conf} \
  /usr/src/sc200pc-0.9.0/

sudo dkms add -m sc200pc -v 0.9.0
sudo dkms build -m sc200pc -v 0.9.0
sudo dkms install -m sc200pc -v 0.9.0
```

Confirm both installed:

```bash
dkms status
# should show:
# ipu-bridge-sslc2000/1.0, <kernel>, x86_64: installed
# sc200pc/0.9.0, <kernel>, x86_64: installed
```

### 2.3 — Support files (tuning YAML + WirePlumber rule)

```bash
sudo install -Dm644 packaging/galaxybook6pro-camera/50-ipu7-hide-v4l2.conf \
  /etc/wireplumber/wireplumber.conf.d/50-ipu7-hide-v4l2.conf

sudo install -Dm644 packaging/galaxybook6pro-camera/sc200pc.yaml \
  /usr/local/share/libcamera/ipa/simple/sc200pc.yaml

sudo install -Dm755 packaging/galaxybook6pro-camera/sc200pc-libcamera-check \
  /usr/local/bin/sc200pc-libcamera-check
```

**Important note about the tuning YAML:** in libcamera 0.7.0, the `Adjust` algorithm (gamma/contrast/saturation) **completely ignores** whatever is in the tuning file — those values are set at runtime via controls (`controls::Gamma`, etc.), not via YAML. The gamma default is already 2.2 (reasonable). Do not add a `Lut:` block to the YAML — that algorithm doesn't exist in this version and produces the error `Algorithm 'Lut' not found`. `BlackLevel: level: N` has no effect either — that value is computed automatically from each frame's histogram. The only parameters in this YAML that actually do anything are `Ccm` (color correction matrices) and `Agc.relativeLuminanceTarget` (default ~0.16; raising it brightens the image but increases noise in low light).

---

## 3. `/dev/dma_heap` permissions (needed for software debayering)

Without this, `cam` enumerates the camera but fails to capture with `Could not open any dma-buf provider`.

```bash
echo 'SUBSYSTEM=="dma_heap", KERNEL=="system", MODE="0660", GROUP="video"' | \
  sudo tee /etc/udev/rules.d/99-dma-heap.rules

sudo usermod -aG video "$USER"

sudo udevadm control --reload-rules
sudo udevadm trigger
```

**You need to log out and back in** (or reboot) for the group change to take effect.

---

## 4. Persist module load order across boots

The sensor module (`sc200pc`) has to load **before** `intel_ipu7`, or the IPU builds its media graph without finding the sensor (`No sensor found for /dev/media0` / `no subdev found in graph`).

```bash
echo "sc200pc" | sudo tee /etc/modules-load.d/sc200pc.conf
```

Verify after a reboot (see section 6 — Final verification).

---

## 5. Rebuild PipeWire's SPA module against libcamera 0.7.0

This step is required for Chrome, Firefox, and any "PipeWire-native" app to see the camera — without it, `cam` works but the camera never appears outside the terminal. Apt's `pipewire-libcamera` package ships its own module linked against the repos' old libcamera 0.2.0, which is incompatible with the `simple` pipeline this sensor needs.

### 5.1 — Remove the competing apt packages

```bash
sudo apt remove --purge pipewire-libcamera libspa-0.2-libcamera libcamera0.2
```

### 5.2 — Build only PipeWire's libcamera SPA module

Use the same PipeWire version already installed on the system (`pipewire --version` to check — tested with 1.6.8).

```bash
sudo apt build-dep pipewire
sudo apt install git meson ninja-build pkg-config

git clone --branch 1.6.8 --depth 1 https://gitlab.freedesktop.org/pipewire/pipewire.git ~/pipewire-src
cd ~/pipewire-src

PKG_CONFIG_PATH=/usr/local/lib/x86_64-linux-gnu/pkgconfig meson setup build \
  -Dlibcamera=enabled \
  -Dsession-managers=[] \
  -Dpipewire-alsa=disabled \
  -Dpipewire-jack=disabled \
  -Dpipewire-v4l2=disabled \
  -Dexamples=disabled \
  -Dtests=disabled \
  -Ddocs=disabled \
  -Dman=disabled

ninja -C build spa/plugins/libcamera/libspa-libcamera.so
```

Confirm it linked against the right libcamera:

```bash
ldd build/spa/plugins/libcamera/libspa-libcamera.so | grep libcamera
# should show /usr/local/lib/x86_64-linux-gnu/libcamera.so.0.7, NOT /lib/x86_64-linux-gnu/...0.2
```

### 5.3 — Install the built module

```bash
sudo mkdir -p /usr/lib/x86_64-linux-gnu/spa-0.2/libcamera
sudo cp ~/pipewire-src/build/spa/plugins/libcamera/libspa-libcamera.so \
  /usr/lib/x86_64-linux-gnu/spa-0.2/libcamera/libspa-libcamera.so
sudo chmod 755 /usr/lib/x86_64-linux-gnu/spa-0.2/libcamera/libspa-libcamera.so
```

### 5.4 — Restart PipeWire and the portals

```bash
systemctl --user restart wireplumber pipewire pipewire-pulse
systemctl --user restart xdg-desktop-portal xdg-desktop-portal-gtk
sleep 2
wpctl status | grep -A8 "Video"
# should list: NN. sc200pc [libcamera], and a "Built-in Front Camera" source
```

---

## 6. Final verification after a full reboot

```bash
sudo reboot
```

After rebooting, **without touching anything manually**:

```bash
lsmod | grep -E "sc200pc|ipu7|ipu_bridge"
sudo dmesg | grep -E "bind sc200pc|All sensor registration"
cam -l
wpctl status | grep -A8 "Video"
```

You should see:
- `sc200pc` loaded before `intel_ipu7` in `lsmod`
- `bind sc200pc ... All sensor registration completed.` in `dmesg`
- `cam -l` listing `Internal front camera` or similar
- an `sc200pc [libcamera]` node in `wpctl status`

---

## 7. Enable Chrome (Firefox works out of the box, no extra step needed)

1. Fully quit Chrome: `pkill -f chrome`
2. Open Chrome, go to `chrome://flags/#enable-webrtc-pipewire-camera`, enable it
3. Relaunch Chrome with the button below the flag
4. **If the camera still isn't detected**, reboot the whole machine once more — desktop portals (`xdg-desktop-portal`) can end up with a stale connection to PipeWire if the PipeWire services were restarted live, and a full system reboot reliably fixes it.

---

## Troubleshooting — issues already seen and their cause

### libcamera version conflicts (`symbol lookup error`)
Symptom: `cam: symbol lookup error: ... undefined symbol: _ZN9libcamera...`
Cause: **two versions** of `libcamera.so`/`libcamera-base.so` end up installed at once under `/usr/local/lib/x86_64-linux-gnu/` (e.g. a `.0.7.0` and a `.0.7.2` from a previous build). The active symlink may point to the old one.
Fix:
```bash
sudo find / -xdev -iname "libcamera*.so*" 2>/dev/null   # look for duplicates
sudo rm -f /usr/local/lib/x86_64-linux-gnu/libcamera*.so.0.7.2   # remove the old one
cd ~/libcamera && sudo ninja -C build install && sudo ldconfig
```

### `cam -l` works but PipeWire/Chrome/Cheese don't see the camera
See section 5 — the PipeWire SPA module needs to be rebuilt against the correct libcamera. Cheese specifically has no simple fix without that step (there's no v4l2loopback shortcut Cheese can use directly for this camera).

### `No sensor found for /dev/media0` / `no subdev found in graph`
Cause: the `sc200pc` module isn't loaded, or it loaded **after** `intel_ipu7`.
Fix:
```bash
sudo modprobe -r intel_ipu7_isys intel_ipu7 ipu_bridge sc200pc
sudo modprobe sc200pc
sudo modprobe intel_ipu7
```
Also confirm `/etc/modules-load.d/sc200pc.conf` exists (section 4) so this doesn't recur on every boot.

### `Could not open any dma-buf provider`
See section 3 — `/dev/dma_heap/system` permissions.

### Washed-out / milky-grey image, low contrast
Not a tuning-YAML issue (see note in section 2.3). Could be:
- Optical flare from a very bright light source in frame — try moving the light out of frame
- Factory protective film still on the physical lens — check visually
- Pending confirmation in daylight (evaluation still in progress as of this writeup)

---

## 8. Audio — no sound from the speakers (CS35L57 / SDCA)

### Initial diagnosis (turned out to be a red herring)

The first symptom visible in `dmesg` is:
```
cs35l56 sdw:0:1:01fa:3557:01: FIRMWARE_MISSING
cs35l56 sdw:0:1:01fa:3557:01: Calibration disabled due to missing firmware controls
```
This looks like the Samsung factory calibration file for the CS35L57 amplifiers (SSID `144dc910`) is missing — a reported, still publicly unresolved issue in `linux-firmware` as of this writing. **However, this is NOT the actual cause of the silence**: confirmed by booting an Ubuntu 26.04 Live USB, which shows the exact same `FIRMWARE_MISSING` line by line but **does** play audio — the factory calibration burned into the chip itself (OTP) is enough to work; what's missing is only Samsung's fine-tuning profile, which is non-blocking.

### Real cause: outdated `alsa-ucm-conf` on Pop!_OS

```bash
sudo alsactl restore
# telling error:
# could not open configuration file /usr/share/alsa/ucm2/sof-soundwire/cs42l45-dmic.conf
```

The `alsa-ucm-conf` package installed by default on Pop!_OS is a System76-specific build (`1.2.10-1ubuntu5.9pop0~...`) that doesn't include the full UCM2 profile for this board's `cs42l45` codec — while Ubuntu's own repo package (`1.2.10-1ubuntu5.14`, in `noble-updates`) does.

### Fix

```bash
sudo apt install alsa-ucm-conf=1.2.10-1ubuntu5.14

# confirm the file now exists:
ls /usr/share/alsa/ucm2/sof-soundwire/ | grep cs42l45
# should list: cs42l45.conf  cs42l45-dmic.conf

sudo alsactl init
sudo alsactl store    # save the healthy (unmuted) state as the one loaded on every boot
```

### Side effect: volume keys stopped responding

During diagnosis (repeated live restarts of `wireplumber`/`pipewire`/`pipewire-pulse`), `cosmic-session` — the process that translates physical volume keys into PipeWire commands — was left with a stale connection to a previous PipeWire instance. Symptom in the logs:
```bash
journalctl --user -f
# ERROR  Failed to raise volume: no active sink device to apply operation to
```
The keys were sending the correct event at the kernel level (confirmable with `evtest` on the "AT Translated Set 2 keyboard" device: `KEY_VOLUMEUP`/`KEY_VOLUMEDOWN` came through fine), but `cosmic-session` couldn't apply them.

**Fix:** restart the full graphical session (log out and back in, or reboot the machine). A plain `systemctl --user restart pipewire` isn't enough — `cosmic-session` needs to relaunch itself to reconnect.

### Lesson for a from-scratch reinstall

After a clean Pop!_OS install, before assuming audio just doesn't work:
1. Update `alsa-ucm-conf` to Ubuntu's `noble-updates` version (command above) — this may prevent the whole issue from the start.
2. If something still ended up inconsistent from manual ALSA/PipeWire testing, a full system reboot (not just restarting services) fixes `cosmic-session`'s state.
3. Don't chase the CS35L57 `FIRMWARE_MISSING` as the cause of the silence — it's expected noise on this hardware, not blocking.

---

## References

- Sensor driver base project: `https://github.com/Jabbslad/sc200pc-linux`
- Original hardware report (Onuralp Akca, linux-media, Aug. 2026): confirms the ACPI HID `SSLC2000` and the SC200PC sensor
- libcamera documentation: `https://libcamera.org`
- Exact laptop model: `NP940XJG-LG1BR` (Samsung Galaxy Book6 Pro, Brazil version)
