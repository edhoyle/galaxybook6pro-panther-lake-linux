# Habilitar cámara y audio del Samsung Galaxy Book6 Pro en Pop!_OS — Instalación desde cero

[English](README.md) | [Português](README.pt-BR.md)

**Hardware:** Samsung Galaxy Book6 Pro (Panther Lake), modelo `NP940XJG-LG1BR`.
- Cámara: sensor Samsung/SmartSens SC200PC, ACPI HID `SSLC2000`, sobre Intel IPU7.
- Audio: amplificadores de parlantes Cirrus Logic CS35L57 (x2) vía SoundWire/SDCA, códec de auriculares/mic cs42l45.

**Problema base cámara:** ni el kernel ni libcamera reconocen este sensor de forma nativa. Se necesita: (1) una libcamera lo bastante reciente para el pipeline `simple`/SoftISP, (2) un driver de sensor y un parche de `ipu-bridge` fuera de árbol (proyecto `sc200pc-linux`), y (3) un módulo de PipeWire recompilado contra esa libcamera para que las apps normales (Chrome, Firefox) vean la cámara.

**Problema base audio:** el paquete `alsa-ucm-conf` que trae Pop!_OS (variante System76) está desactualizado respecto al de Ubuntu y le falta el perfil UCM2 del códec `cs42l45` para esta placa — sin ese archivo, WirePlumber puede dejar el sink de parlantes en un estado inconsistente (muteado / sin ruta activa) aunque el hardware y el firmware SOF carguen bien. Ver sección 8.

**Versión de Pop!_OS usada:** Pop!_OS 24.04 LTS (COSMIC), base Ubuntu `noble`, kernel `7.1.5-76070105-generic`. Confirmá la tuya con `cat /etc/os-release` o `pop-os-select`.

Esta guía asume una instalación limpia de Pop!_OS. Seguir los pasos en orden — cada sección depende de la anterior.

---

## 0. Antes de empezar

- Confirmar la versión del kernel: `uname -r`. Se probó con `7.1.5-76070105-generic`. Versiones de kernel muy distintas pueden requerir volver a compilar los módulos DKMS (se reconstruyen solos si `dkms` está bien configurado).
- Todo lo que sigue asume una sesión Wayland (`echo $XDG_SESSION_TYPE` → `wayland`), que es el default en Pop!_OS con COSMIC.

---

## 1. Compilar libcamera 0.7.0 desde el código fuente

Pop!_OS (base Ubuntu/noble) solo trae `libcamera-ipa 0.2.0` en sus repos — muy vieja para el pipeline `simple` que necesita este sensor. Hay que compilar libcamera a mano.

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

Confirmar que se instaló limpio:

```bash
cam -l
```

Debería listar `Available cameras:` sin errores de símbolos (`symbol lookup error`). Si aparece ese error, ver la sección **Troubleshooting → Conflictos de versión de libcamera** más abajo — es el problema más común de todo este proceso.

**Nota sobre libgles2-mesa-dev:** es necesario porque el SoftISP usa aceleración por GPU (EGL/GLES2) para el debayering. Si se prefiere evitar esa dependencia, se puede compilar con `-Dsoftisp-gpu-accel=disabled`, a costa de más uso de CPU (~63% de un core vs. ~9% con GPU, según mediciones del proyecto de referencia).

---

## 2. Instalar los módulos DKMS del sensor (proyecto `sc200pc-linux`)

El proyecto de referencia (empaquetado originalmente para Arch Linux) vive en:
`https://github.com/Jabbslad/sc200pc-linux`

Como está empaquetado para AUR/`makepkg`, en Pop!_OS hay que extraer el contenido real y armar los paquetes DKMS a mano.

```bash
sudo apt install dkms

git clone https://github.com/Jabbslad/sc200pc-linux ~/sc200pc-linux
cd ~/sc200pc-linux
```

### 2.1 — Módulo `ipu-bridge-sslc2000` (parche del kernel que reconoce el sensor)

```bash
sudo mkdir -p /usr/src/ipu-bridge-sslc2000-1.0
sudo cp packaging/ipu-bridge-sslc2000-dkms/{Kbuild,Makefile,ipu-bridge.c,dkms.conf} \
  /usr/src/ipu-bridge-sslc2000-1.0/
sudo sed -i 's/@PKGVER@/1.0/' /usr/src/ipu-bridge-sslc2000-1.0/dkms.conf

sudo dkms add -m ipu-bridge-sslc2000 -v 1.0
sudo dkms build -m ipu-bridge-sslc2000 -v 1.0
sudo dkms install -m ipu-bridge-sslc2000 -v 1.0
```

### 2.2 — Módulo `sc200pc` (driver V4L2 del sensor)

```bash
sudo mkdir -p /usr/src/sc200pc-0.9.0
sudo cp packaging/sc200pc-dkms/{Kbuild,Makefile,sc200pc.c,dkms.conf} \
  /usr/src/sc200pc-0.9.0/

sudo dkms add -m sc200pc -v 0.9.0
sudo dkms build -m sc200pc -v 0.9.0
sudo dkms install -m sc200pc -v 0.9.0
```

Confirmar que ambos quedaron instalados:

```bash
dkms status
# debe mostrar:
# ipu-bridge-sslc2000/1.0, <kernel>, x86_64: installed
# sc200pc/0.9.0, <kernel>, x86_64: installed
```

### 2.3 — Archivos de soporte (tuning YAML + regla de WirePlumber)

```bash
sudo install -Dm644 packaging/galaxybook6pro-camera/50-ipu7-hide-v4l2.conf \
  /etc/wireplumber/wireplumber.conf.d/50-ipu7-hide-v4l2.conf

sudo install -Dm644 packaging/galaxybook6pro-camera/sc200pc.yaml \
  /usr/local/share/libcamera/ipa/simple/sc200pc.yaml

sudo install -Dm755 packaging/galaxybook6pro-camera/sc200pc-libcamera-check \
  /usr/local/bin/sc200pc-libcamera-check
```

**Nota importante sobre el YAML de tuning:** en libcamera 0.7.0, el algoritmo `Adjust` (gamma/contraste/saturación) **ignora por completo** lo que haya en el tuning file — esos valores se setean en runtime vía controles (`controls::Gamma`, etc.), no por YAML. El default de gamma ya es 2.2 (razonable). No agregar un bloque `Lut:` al YAML — ese algoritmo no existe en esta versión y produce el error `Algorithm 'Lut' not found`. Tampoco tiene efecto `BlackLevel: level: N` — ese valor se calcula solo, del histograma de cada frame. Los únicos parámetros de este YAML que sí tienen efecto real son los de `Ccm` (matrices de corrección de color) y `Agc.relativeLuminanceTarget` (default ~0.16; subirlo aclara la imagen pero sube el ruido en poca luz).

---

## 3. Permisos de `/dev/dma_heap` (necesario para el debayering por software)

Sin esto, `cam` enumera la cámara pero falla al capturar con `Could not open any dma-buf provider`.

```bash
echo 'SUBSYSTEM=="dma_heap", KERNEL=="system", MODE="0660", GROUP="video"' | \
  sudo tee /etc/udev/rules.d/99-dma-heap.rules

sudo usermod -aG video "$USER"

sudo udevadm control --reload-rules
sudo udevadm trigger
```

**Hay que cerrar sesión y volver a entrar** (o reiniciar) para que el cambio de grupo tome efecto.

---

## 4. Persistir el orden de carga de módulos en el boot

El sensor (`sc200pc`) tiene que cargar **antes** que `intel_ipu7`, o el IPU arma el grafo de medios sin encontrar el sensor (`No sensor found for /dev/media0` / `no subdev found in graph`).

```bash
echo "sc200pc" | sudo tee /etc/modules-load.d/sc200pc.conf
```

Verificar tras un reinicio (ver sección 6 — Verificación final).

---

## 5. Recompilar el módulo SPA de PipeWire contra la libcamera 0.7.0

Este paso es imprescindible para que Chrome, Firefox y cualquier app "PipeWire-nativa" vean la cámara — sin esto, `cam` funciona pero la cámara no aparece fuera de la terminal. El paquete `pipewire-libcamera` de apt trae su propio módulo enlazado contra la libcamera 0.2.0 vieja de los repos, incompatible con el pipeline `simple` que necesita este sensor.

### 5.1 — Sacar los paquetes de apt que van a competir

```bash
sudo apt remove --purge pipewire-libcamera libspa-0.2-libcamera libcamera0.2
```

### 5.2 — Compilar solo el módulo SPA de libcamera de PipeWire

Usar la misma versión de PipeWire que ya está instalada en el sistema (`pipewire --version` para confirmar cuál es — se probó con 1.6.8).

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

Confirmar que quedó enlazado contra la libcamera correcta:

```bash
ldd build/spa/plugins/libcamera/libspa-libcamera.so | grep libcamera
# debe mostrar /usr/local/lib/x86_64-linux-gnu/libcamera.so.0.7, NO /lib/x86_64-linux-gnu/...0.2
```

### 5.3 — Instalar el módulo compilado

```bash
sudo mkdir -p /usr/lib/x86_64-linux-gnu/spa-0.2/libcamera
sudo cp ~/pipewire-src/build/spa/plugins/libcamera/libspa-libcamera.so \
  /usr/lib/x86_64-linux-gnu/spa-0.2/libcamera/libspa-libcamera.so
sudo chmod 755 /usr/lib/x86_64-linux-gnu/spa-0.2/libcamera/libspa-libcamera.so
```

### 5.4 — Reiniciar PipeWire y los portales

```bash
systemctl --user restart wireplumber pipewire pipewire-pulse
systemctl --user restart xdg-desktop-portal xdg-desktop-portal-gtk
sleep 2
wpctl status | grep -A8 "Video"
# debe listar: NN. sc200pc [libcamera], y una fuente "Built-in Front Camera"
```

---

## 6. Verificación final tras un reinicio completo

```bash
sudo reboot
```

Después del reinicio, **sin tocar nada manualmente**:

```bash
lsmod | grep -E "sc200pc|ipu7|ipu_bridge"
sudo dmesg | grep -E "bind sc200pc|All sensor registration"
cam -l
wpctl status | grep -A8 "Video"
```

Se debe ver:
- `sc200pc` cargado antes que `intel_ipu7` en `lsmod`
- `bind sc200pc ... All sensor registration completed.` en `dmesg`
- `cam -l` listando `Internal front camera` o similar
- un nodo `sc200pc [libcamera]` en `wpctl status`

---

## 7. Habilitar Chrome (Firefox funciona directo, sin este paso)

1. Cerrar Chrome del todo: `pkill -f chrome`
2. Abrir Chrome, ir a `chrome://flags/#enable-webrtc-pipewire-camera`, activar (Enabled)
3. Reiniciar Chrome con el botón que aparece abajo del flag
4. **Si sigue sin detectar la cámara**, reiniciar toda la máquina una vez más — los portales de escritorio (`xdg-desktop-portal`) a veces quedan con una conexión a PipeWire rota si se reiniciaron los servicios de PipeWire en caliente, y un reinicio completo del sistema lo soluciona de forma confiable.

---

## Troubleshooting — problemas ya vistos y su causa

### Conflictos de versión de libcamera (`symbol lookup error`)
Síntoma: `cam: symbol lookup error: ... undefined symbol: _ZN9libcamera...`
Causa: quedan **dos versiones** de `libcamera.so`/`libcamera-base.so` instaladas a la vez en `/usr/local/lib/x86_64-linux-gnu/` (por ejemplo una `.0.7.0` y una `.0.7.2` de un build anterior). El símlink activo puede apuntar a la vieja.
Solución:
```bash
sudo find / -xdev -iname "libcamera*.so*" 2>/dev/null   # buscar duplicados
sudo rm -f /usr/local/lib/x86_64-linux-gnu/libcamera*.so.0.7.2   # borrar la vieja
cd ~/libcamera && sudo ninja -C build install && sudo ldconfig
```

### `cam -l` funciona pero PipeWire/Chrome/Cheese no ven la cámara
Ver sección 5 — falta recompilar el módulo SPA de PipeWire contra la libcamera correcta. Cheese específicamente no tiene solución simple sin ese paso (no hay atajo vía v4l2loopback que Cheese pueda usar directamente para esta cámara).

### `No sensor found for /dev/media0` / `no subdev found in graph`
Causa: el módulo `sc200pc` no está cargado, o cargó **después** que `intel_ipu7`.
Solución:
```bash
sudo modprobe -r intel_ipu7_isys intel_ipu7 ipu_bridge sc200pc
sudo modprobe sc200pc
sudo modprobe intel_ipu7
```
Y confirmar que `/etc/modules-load.d/sc200pc.conf` existe (sección 4) para que esto no se repita en cada boot.

### `Could not open any dma-buf provider`
Ver sección 3 — permisos de `/dev/dma_heap/system`.

### Imagen "lavada" / gris lechosa, bajo contraste
No es un problema de tuning YAML (ver nota en sección 2.3). Puede ser:
- Flare óptico por una fuente de luz muy brillante en cuadro — probar con la luz fuera de encuadre
- Película protectora de fábrica todavía puesta sobre el lente físico — revisar a simple vista
- Pendiente de confirmar con luz de día (evaluación en curso al momento de escribir este informe)

---

## 8. Audio — parlantes sin sonido (CS35L57 / SDCA)

### Diagnóstico inicial (resultó ser una pista falsa)

El primer síntoma visible en `dmesg` es:
```
cs35l56 sdw:0:1:01fa:3557:01: FIRMWARE_MISSING
cs35l56 sdw:0:1:01fa:3557:01: Calibration disabled due to missing firmware controls
```
Esto parece indicar que falta el archivo de calibración de fábrica de Samsung para los amplificadores CS35L57 (SSID `144dc910`), un problema reportado y sin resolución pública en `linux-firmware` al momento de escribir esto. **Sin embargo, esto NO es la causa del silencio real**: se confirmó arrancando un Live USB de Ubuntu 26.04, que muestra el mismo `FIRMWARE_MISSING` línea por línea pero **sí reproduce audio** — la calibración de fábrica quemada en el propio chip (OTP) alcanza para funcionar; lo que falta es solo el perfil de ajuste fino de Samsung, no bloqueante.

### Causa real: `alsa-ucm-conf` desactualizado en Pop!_OS

```bash
sudo alsactl restore
# error revelador:
# could not open configuration file /usr/share/alsa/ucm2/sof-soundwire/cs42l45-dmic.conf
```

El paquete `alsa-ucm-conf` instalado por defecto en Pop!_OS es una variante propia de System76 (`1.2.10-1ubuntu5.9pop0~...`) que no incluye el perfil UCM2 completo para el códec `cs42l45` de esta placa — mientras que el paquete estándar de los repos de Ubuntu (`1.2.10-1ubuntu5.14`, en `noble-updates`) sí lo trae.

### Solución

```bash
sudo apt install alsa-ucm-conf=1.2.10-1ubuntu5.14

# confirmar que el archivo ahora existe:
ls /usr/share/alsa/ucm2/sof-soundwire/ | grep cs42l45
# debe listar: cs42l45.conf  cs42l45-dmic.conf

sudo alsactl init
sudo alsactl store    # guarda el estado sano (sin mute) como el que se carga en cada boot
```

### Efecto secundario: teclas de volumen dejaron de responder

Durante el diagnóstico (reinicios en caliente repetidos de `wireplumber`/`pipewire`/`pipewire-pulse`), `cosmic-session` — el proceso que traduce las teclas físicas de volumen a comandos de PipeWire — quedó con una conexión obsoleta a una instancia anterior de PipeWire. Síntoma en los logs:
```bash
journalctl --user -f
# ERROR  Failed to raise volume: no active sink device to apply operation to
```
Las teclas mandaban el evento correcto a nivel kernel (confirmable con `evtest` sobre el dispositivo "AT Translated Set 2 keyboard": `KEY_VOLUMEUP`/`KEY_VOLUMEDOWN` llegaban bien), pero `cosmic-session` no los podía aplicar.

**Solución:** reiniciar la sesión gráfica completa (cerrar sesión y volver a entrar, o reiniciar el equipo). Un simple `systemctl --user restart pipewire` no alcanza — `cosmic-session` necesita relanzarse él mismo para reconectar.

### Lección para una reinstalación desde cero

Después de instalar Pop!_OS limpio, antes de dar por sentado que el audio no funciona:
1. Actualizar `alsa-ucm-conf` a la versión de `noble-updates` de Ubuntu (comando arriba) — esto puede evitar el problema por completo desde el principio.
2. Si de todas formas quedó algo inconsistente por pruebas manuales de ALSA/PipeWire, un reinicio completo del sistema (no solo de los servicios) resuelve el estado de `cosmic-session`.
3. No perseguir el `FIRMWARE_MISSING` del CS35L57 como causa del silencio — es ruido esperado en este hardware, no bloqueante.

---

## Referencias

- Proyecto base del driver del sensor: `https://github.com/Jabbslad/sc200pc-linux`
- Reporte original del hardware (Onuralp Akca, linux-media, ago. 2026): confirma el HID ACPI `SSLC2000` y el sensor SC200PC
- Documentación de libcamera: `https://libcamera.org`
- Modelo exacto de la laptop: `NP940XJG-LG1BR` (Samsung Galaxy Book6 Pro, versión Brasil)
