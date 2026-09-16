# Habilitando câmera e áudio no Samsung Galaxy Book6 Pro com Pop!_OS — instalação do zero

[English](README.md) | [Español](README.es.md)

**Hardware:** Samsung Galaxy Book6 Pro (Panther Lake), modelo `NP940XJG-LG1BR`.
- Câmera: sensor Samsung/SmartSens SC200PC, ACPI HID `SSLC2000`, sobre Intel IPU7.
- Áudio: amplificadores de alto-falantes Cirrus Logic CS35L57 (x2) via SoundWire/SDCA, codec de fone/microfone cs42l45.

**Problema base da câmera:** nem o kernel nem o libcamera reconhecem esse sensor de forma nativa. É necessário: (1) um libcamera recente o suficiente para o pipeline `simple`/SoftISP, (2) um driver de sensor e um patch do `ipu-bridge` fora da árvore do kernel (projeto `sc200pc-linux`), e (3) um módulo do PipeWire recompilado contra esse libcamera para que apps normais (Chrome, Firefox) enxerguem a câmera.

**Problema base do áudio:** o pacote `alsa-ucm-conf` que vem com o Pop!_OS (variante System76) está desatualizado em relação ao do Ubuntu e falta o perfil UCM2 do codec `cs42l45` desta placa — sem esse arquivo, o WirePlumber pode deixar o sink dos alto-falantes em um estado inconsistente (mutado / sem rota ativa) mesmo com o hardware e o firmware SOF carregando corretamente. Ver seção 8.

**Versão do Pop!_OS usada:** Pop!_OS 24.04 LTS (COSMIC), base Ubuntu `noble`, kernel `7.1.5-76070105-generic`. Confirme a sua com `cat /etc/os-release` ou `pop-os-select`.

Este guia assume uma instalação limpa do Pop!_OS. Siga os passos em ordem — cada seção depende da anterior.

---

## 0. Antes de começar

- Confirme a versão do kernel: `uname -r`. Testado com `7.1.5-76070105-generic`. Versões de kernel muito diferentes podem exigir recompilar os módulos DKMS (eles se reconstroem sozinhos se o `dkms` estiver bem configurado).
- Tudo a seguir assume uma sessão Wayland (`echo $XDG_SESSION_TYPE` → `wayland`), o padrão no Pop!_OS com COSMIC.

---

## 1. Compilar o libcamera 0.7.0 a partir do código-fonte

O Pop!_OS (base Ubuntu/noble) traz apenas `libcamera-ipa 0.2.0` em seus repositórios — muito antigo para o pipeline `simple` que este sensor precisa. É necessário compilar o libcamera manualmente.

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

Confirme que instalou corretamente:

```bash
cam -l
```

Deve listar `Available cameras:` sem erros de símbolo (`symbol lookup error`). Se esse erro aparecer, veja a seção **Troubleshooting → Conflitos de versão do libcamera** mais abaixo — é o problema mais comum de todo esse processo.

**Nota sobre libgles2-mesa-dev:** necessário porque o SoftISP usa aceleração por GPU (EGL/GLES2) para o debayering. Para evitar essa dependência, é possível compilar com `-Dsoftisp-gpu-accel=disabled`, ao custo de mais uso de CPU (~63% de um núcleo vs. ~9% com GPU, segundo as medições do projeto de referência).

---

## 2. Instalar os módulos DKMS do sensor (projeto `sc200pc-linux`)

O projeto de referência (originalmente empacotado para Arch Linux) está em:
`https://github.com/Jabbslad/sc200pc-linux`

Como está empacotado para AUR/`makepkg`, no Pop!_OS é preciso extrair o conteúdo real e montar os pacotes DKMS manualmente.

```bash
sudo apt install dkms

git clone https://github.com/Jabbslad/sc200pc-linux ~/sc200pc-linux
cd ~/sc200pc-linux
```

### 2.1 — Módulo `ipu-bridge-sslc2000` (patch do kernel que reconhece o sensor)

```bash
sudo mkdir -p /usr/src/ipu-bridge-sslc2000-1.0
sudo cp packaging/ipu-bridge-sslc2000-dkms/{Kbuild,Makefile,ipu-bridge.c,dkms.conf} \
  /usr/src/ipu-bridge-sslc2000-1.0/
sudo sed -i 's/@PKGVER@/1.0/' /usr/src/ipu-bridge-sslc2000-1.0/dkms.conf

sudo dkms add -m ipu-bridge-sslc2000 -v 1.0
sudo dkms build -m ipu-bridge-sslc2000 -v 1.0
sudo dkms install -m ipu-bridge-sslc2000 -v 1.0
```

### 2.2 — Módulo `sc200pc` (driver V4L2 do sensor)

```bash
sudo mkdir -p /usr/src/sc200pc-0.9.0
sudo cp packaging/sc200pc-dkms/{Kbuild,Makefile,sc200pc.c,dkms.conf} \
  /usr/src/sc200pc-0.9.0/

sudo dkms add -m sc200pc -v 0.9.0
sudo dkms build -m sc200pc -v 0.9.0
sudo dkms install -m sc200pc -v 0.9.0
```

Confirme que os dois ficaram instalados:

```bash
dkms status
# deve mostrar:
# ipu-bridge-sslc2000/1.0, <kernel>, x86_64: installed
# sc200pc/0.9.0, <kernel>, x86_64: installed
```

### 2.3 — Arquivos de suporte (tuning YAML + regra do WirePlumber)

```bash
sudo install -Dm644 packaging/galaxybook6pro-camera/50-ipu7-hide-v4l2.conf \
  /etc/wireplumber/wireplumber.conf.d/50-ipu7-hide-v4l2.conf

sudo install -Dm644 packaging/galaxybook6pro-camera/sc200pc.yaml \
  /usr/local/share/libcamera/ipa/simple/sc200pc.yaml

sudo install -Dm755 packaging/galaxybook6pro-camera/sc200pc-libcamera-check \
  /usr/local/bin/sc200pc-libcamera-check
```

**Nota importante sobre o YAML de tuning:** no libcamera 0.7.0, o algoritmo `Adjust` (gama/contraste/saturação) **ignora completamente** o que estiver no arquivo de tuning — esses valores são definidos em tempo de execução via controles (`controls::Gamma`, etc.), não pelo YAML. O gama padrão já é 2.2 (razoável). Não adicione um bloco `Lut:` ao YAML — esse algoritmo não existe nesta versão e produz o erro `Algorithm 'Lut' not found`. `BlackLevel: level: N` também não tem efeito — esse valor é calculado sozinho, a partir do histograma de cada quadro. Os únicos parâmetros deste YAML que realmente têm efeito são os de `Ccm` (matrizes de correção de cor) e `Agc.relativeLuminanceTarget` (padrão ~0.16; aumentá-lo clareia a imagem, mas aumenta o ruído com pouca luz).

---

## 3. Permissões de `/dev/dma_heap` (necessário para o debayering por software)

Sem isso, o `cam` enumera a câmera, mas falha ao capturar com `Could not open any dma-buf provider`.

```bash
echo 'SUBSYSTEM=="dma_heap", KERNEL=="system", MODE="0660", GROUP="video"' | \
  sudo tee /etc/udev/rules.d/99-dma-heap.rules

sudo usermod -aG video "$USER"

sudo udevadm control --reload-rules
sudo udevadm trigger
```

**É preciso sair da sessão e entrar novamente** (ou reiniciar) para que a mudança de grupo tenha efeito.

---

## 4. Persistir a ordem de carregamento dos módulos no boot

O módulo do sensor (`sc200pc`) precisa carregar **antes** do `intel_ipu7`, ou o IPU monta o grafo de mídia sem encontrar o sensor (`No sensor found for /dev/media0` / `no subdev found in graph`).

```bash
echo "sc200pc" | sudo tee /etc/modules-load.d/sc200pc.conf
```

Verifique após um reboot (ver seção 6 — Verificação final).

---

## 5. Recompilar o módulo SPA do PipeWire contra o libcamera 0.7.0

Este passo é indispensável para que Chrome, Firefox e qualquer app "nativa do PipeWire" enxerguem a câmera — sem ele, o `cam` funciona, mas a câmera não aparece fora do terminal. O pacote `pipewire-libcamera` do apt traz seu próprio módulo vinculado ao libcamera 0.2.0 antigo dos repositórios, incompatível com o pipeline `simple` que este sensor precisa.

### 5.1 — Remover os pacotes do apt que vão competir

```bash
sudo apt remove --purge pipewire-libcamera libspa-0.2-libcamera libcamera0.2
```

### 5.2 — Compilar apenas o módulo SPA de libcamera do PipeWire

Use a mesma versão do PipeWire já instalada no sistema (`pipewire --version` para confirmar qual é — testado com 1.6.8).

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

Confirme que ficou vinculado ao libcamera correto:

```bash
ldd build/spa/plugins/libcamera/libspa-libcamera.so | grep libcamera
# deve mostrar /usr/local/lib/x86_64-linux-gnu/libcamera.so.0.7, NÃO /lib/x86_64-linux-gnu/...0.2
```

### 5.3 — Instalar o módulo compilado

```bash
sudo mkdir -p /usr/lib/x86_64-linux-gnu/spa-0.2/libcamera
sudo cp ~/pipewire-src/build/spa/plugins/libcamera/libspa-libcamera.so \
  /usr/lib/x86_64-linux-gnu/spa-0.2/libcamera/libspa-libcamera.so
sudo chmod 755 /usr/lib/x86_64-linux-gnu/spa-0.2/libcamera/libspa-libcamera.so
```

### 5.4 — Reiniciar o PipeWire e os portais

```bash
systemctl --user restart wireplumber pipewire pipewire-pulse
systemctl --user restart xdg-desktop-portal xdg-desktop-portal-gtk
sleep 2
wpctl status | grep -A8 "Video"
# deve listar: NN. sc200pc [libcamera], e uma fonte "Built-in Front Camera"
```

---

## 6. Verificação final após um reboot completo

```bash
sudo reboot
```

Após reiniciar, **sem mexer em nada manualmente**:

```bash
lsmod | grep -E "sc200pc|ipu7|ipu_bridge"
sudo dmesg | grep -E "bind sc200pc|All sensor registration"
cam -l
wpctl status | grep -A8 "Video"
```

Você deve ver:
- `sc200pc` carregado antes de `intel_ipu7` no `lsmod`
- `bind sc200pc ... All sensor registration completed.` no `dmesg`
- `cam -l` listando `Internal front camera` ou similar
- um nó `sc200pc [libcamera]` no `wpctl status`

---

## 7. Habilitar o Chrome (o Firefox funciona direto, sem este passo)

1. Fechar o Chrome completamente: `pkill -f chrome`
2. Abrir o Chrome, ir em `chrome://flags/#enable-webrtc-pipewire-camera`, ativar (Enabled)
3. Reiniciar o Chrome com o botão que aparece abaixo do flag
4. **Se ainda assim não detectar a câmera**, reinicie a máquina toda mais uma vez — os portais de desktop (`xdg-desktop-portal`) às vezes ficam com uma conexão quebrada com o PipeWire se os serviços do PipeWire foram reiniciados "a quente", e um reboot completo do sistema resolve isso de forma confiável.

---

## Troubleshooting — problemas já vistos e sua causa

### Conflitos de versão do libcamera (`symbol lookup error`)
Sintoma: `cam: symbol lookup error: ... undefined symbol: _ZN9libcamera...`
Causa: ficam **duas versões** de `libcamera.so`/`libcamera-base.so` instaladas ao mesmo tempo em `/usr/local/lib/x86_64-linux-gnu/` (por exemplo uma `.0.7.0` e uma `.0.7.2` de um build anterior). O symlink ativo pode apontar para a antiga.
Solução:
```bash
sudo find / -xdev -iname "libcamera*.so*" 2>/dev/null   # procurar duplicatas
sudo rm -f /usr/local/lib/x86_64-linux-gnu/libcamera*.so.0.7.2   # apagar a antiga
cd ~/libcamera && sudo ninja -C build install && sudo ldconfig
```

### `cam -l` funciona, mas PipeWire/Chrome/Cheese não veem a câmera
Ver seção 5 — falta recompilar o módulo SPA do PipeWire contra o libcamera correto. O Cheese especificamente não tem solução simples sem esse passo (não há atalho via v4l2loopback que o Cheese consiga usar diretamente para esta câmera).

### `No sensor found for /dev/media0` / `no subdev found in graph`
Causa: o módulo `sc200pc` não está carregado, ou carregou **depois** do `intel_ipu7`.
Solução:
```bash
sudo modprobe -r intel_ipu7_isys intel_ipu7 ipu_bridge sc200pc
sudo modprobe sc200pc
sudo modprobe intel_ipu7
```
E confirme que `/etc/modules-load.d/sc200pc.conf` existe (seção 4) para que isso não se repita a cada boot.

### `Could not open any dma-buf provider`
Ver seção 3 — permissões de `/dev/dma_heap/system`.

### Imagem "lavada" / cinza leitosa, baixo contraste
Não é um problema do YAML de tuning (ver nota na seção 2.3). Pode ser:
- Flare óptico causado por uma fonte de luz muito forte no enquadramento — tente tirar a luz do quadro
- Película protetora de fábrica ainda colocada sobre a lente física — verifique a olho nu
- Pendente de confirmação com luz do dia (avaliação em andamento no momento em que este documento foi escrito)

---

## 8. Áudio — alto-falantes sem som (CS35L57 / SDCA)

### Diagnóstico inicial (acabou sendo uma pista falsa)

O primeiro sintoma visível no `dmesg` é:
```
cs35l56 sdw:0:1:01fa:3557:01: FIRMWARE_MISSING
cs35l56 sdw:0:1:01fa:3557:01: Calibration disabled due to missing firmware controls
```
Isso parece indicar que falta o arquivo de calibração de fábrica da Samsung para os amplificadores CS35L57 (SSID `144dc910`), um problema reportado e ainda sem resolução pública no `linux-firmware` até o momento desta redação. **No entanto, isso NÃO é a causa real do silêncio**: confirmado inicializando um Live USB do Ubuntu 26.04, que mostra o mesmo `FIRMWARE_MISSING` linha por linha, mas **reproduz áudio normalmente** — a calibração de fábrica gravada no próprio chip (OTP) já é suficiente para funcionar; o que falta é apenas o perfil de ajuste fino da Samsung, que não é bloqueante.

### Causa real: `alsa-ucm-conf` desatualizado no Pop!_OS

```bash
sudo alsactl restore
# erro revelador:
# could not open configuration file /usr/share/alsa/ucm2/sof-soundwire/cs42l45-dmic.conf
```

O pacote `alsa-ucm-conf` instalado por padrão no Pop!_OS é uma variante própria da System76 (`1.2.10-1ubuntu5.9pop0~...`) que não inclui o perfil UCM2 completo para o codec `cs42l45` desta placa — enquanto o pacote padrão dos repositórios do Ubuntu (`1.2.10-1ubuntu5.14`, em `noble-updates`) traz esse perfil completo.

### Solução

```bash
sudo apt install alsa-ucm-conf=1.2.10-1ubuntu5.14

# confirmar que o arquivo agora existe:
ls /usr/share/alsa/ucm2/sof-soundwire/ | grep cs42l45
# deve listar: cs42l45.conf  cs42l45-dmic.conf

sudo alsactl init
sudo alsactl store    # salva o estado saudável (sem mudo) como o que é carregado a cada boot
```

### Efeito colateral: as teclas de volume pararam de responder

Durante o diagnóstico (reinícios "a quente" repetidos de `wireplumber`/`pipewire`/`pipewire-pulse`), o `cosmic-session` — o processo que traduz as teclas físicas de volume em comandos do PipeWire — ficou com uma conexão obsoleta a uma instância anterior do PipeWire. Sintoma nos logs:
```bash
journalctl --user -f
# ERROR  Failed to raise volume: no active sink device to apply operation to
```
As teclas enviavam o evento correto no nível do kernel (confirmável com `evtest` no dispositivo "AT Translated Set 2 keyboard": `KEY_VOLUMEUP`/`KEY_VOLUMEDOWN` chegavam normalmente), mas o `cosmic-session` não conseguia aplicá-los.

**Solução:** reiniciar a sessão gráfica completa (sair e entrar novamente, ou reiniciar o computador). Um simples `systemctl --user restart pipewire` não é suficiente — o `cosmic-session` precisa se relançar sozinho para reconectar.

### Lição para uma reinstalação do zero

Depois de instalar o Pop!_OS limpo, antes de assumir que o áudio simplesmente não funciona:
1. Atualize o `alsa-ucm-conf` para a versão do `noble-updates` do Ubuntu (comando acima) — isso pode evitar o problema completamente desde o início.
2. Se mesmo assim algo ficar inconsistente por causa de testes manuais de ALSA/PipeWire, um reboot completo do sistema (não apenas reiniciar os serviços) resolve o estado do `cosmic-session`.
3. Não persiga o `FIRMWARE_MISSING` do CS35L57 como causa do silêncio — é um ruído esperado neste hardware, não é bloqueante.

---

## Referências

- Projeto base do driver do sensor: `https://github.com/Jabbslad/sc200pc-linux`
- Relatório original do hardware (Onuralp Akca, linux-media, ago. 2026): confirma o HID ACPI `SSLC2000` e o sensor SC200PC
- Documentação do libcamera: `https://libcamera.org`
- Modelo exato do notebook: `NP940XJG-LG1BR` (Samsung Galaxy Book6 Pro, versão Brasil)
