# Plataforma de Projeção Sem Fio para Ambientes Educacionais

Protótipo de receptor de projeção sem fio baseado no reaproveitamento de TV Boxes
apreendidas, rodando Armbian Linux. Parte do **Projeto Transformar**, parceria entre o
IFPR Campus Capanema, a UTFPR Campus Francisco Beltrão e a Receita Federal do Brasil.

> **Status: trabalho em andamento (work in progress).** O receptor Miracast funciona
> de ponta a ponta com dispositivos Android e Windows, mas ainda apresenta
> instabilidade em hardware limitado (delay de 1–2s, quedas de FPS, e ocasionalmente
> falha em exibir a imagem). Um caminho alternativo, sem Wi-Fi Direct/Miracast, também
> está documentado abaixo e tem latência bem menor.

Ideia original inspirada no trabalho de
[Guilherme Sotti Neto — tv-box](https://github.com/guilherme-sotti-neto/tv-box)
(reaproveitamento de TV Box com Armbian), estendida aqui para um receptor de
projeção sem fio via Wi-Fi Direct/Miracast.

## Hardware do protótipo

| Componente | Valor confirmado |
|---|---|
| Modelo | TV Box BTV B11 |
| SoC | Amlogic S905X3 (ARM Cortex-A55, quad-core, 1.0–2.1 GHz) — confirmado via `lscpu`/MIDR |
| DTB em uso | Device tree de um S905X (não X3) — necessário empiricamente: qualquer outro DTB testado falha ao iniciar |
| RAM | 944 MiB total (DDR3) — ~376 MiB em uso / 568 MiB livres em repouso, com Xorg+i3 ativos |
| Armazenamento | eMMC, 14,6 GiB utilizáveis |
| GPU / VPU | Mali-G31 (driver Panfrost) + VPU Amlogic Meson (`meson-drm`) |
| Wi-Fi / Bluetooth | Broadcom BCM43430A1 (módulo combo AP6212A1), Wi-Fi Direct confirmado (P2P-client/GO/device) |
| SO | Armbian Linux (aarch64) |

## Arquitetura

```
Dispositivo do docente  --Wi-Fi Direct/Miracast-->  TV Box (Armbian)  --HDMI-->  Projetor
      (Windows/Android)         (RTSP + H.264)         (miraclecast + GStreamer)
```

## Caminho 1 — Wi-Fi Direct / Miracast (objetivo principal)

Usa a implementação livre [MiracleCast](https://github.com/albfan/miraclecast)
(`miracle-wifid` + `miracle-sinkctl`), compilada a partir da branch `wip/windows-fix`.

### Pré-requisitos
```bash
apt install cmake build-essential git pkg-config libudev-dev libglib2.0-dev \
  libsystemd-dev check libreadline-dev libtool autoconf-archive \
  gstreamer1.0-tools gstreamer1.0-plugins-base gstreamer1.0-plugins-good \
  gstreamer1.0-plugins-bad gstreamer1.0-x xserver-xorg xinit i3
```

### Compilação
```bash
git clone https://github.com/albfan/miraclecast.git
cd miraclecast
git checkout wip/windows-fix
./autogen.sh
./configure CFLAGS='-g -O0 -ftrapv' --sysconfdir=/etc --localstatedir=/var --libdir=/usr/lib
make -j$(nproc)
make install
```

### Configurações aplicadas (ver pasta `configs/`)
- `configs/99-unmanaged-wlan0.conf` → `/etc/NetworkManager/conf.d/` — mantém o `wlan0`
  sempre fora do NetworkManager, dedicado ao Wi-Fi Direct
- `configs/miracle-wifid.service` → `/etc/systemd/system/` — sobe o daemon de Wi-Fi
  Direct automaticamente no boot
- `configs/miracle-gst` → `/usr/local/bin/miracle-gst` — script de renderização
  ajustado (`xvimagesink`, `sync=false`) para funcionar sobre uma sessão Xorg+i3
  mínima, em vez do `autovideosink`/`kmssink` padrão
- `configs/miraclecastrc` → `~/.miraclecastrc` — extensão de protocolo que responde
  explicitamente aos parâmetros (`wfd_display_edid`, `microsoft_*`) exigidos pelo
  Windows no `GET_PARAMETER`, sem os quais a sessão RTSP trava/desconecta
- `configs/xinitrc` → `~/.xinitrc` — inicia o i3 como gerenciador de janelas mínimo
- `configs/zprofile-snippet.sh` → acrescentar ao `~/.zprofile` — inicia o Xorg
  automaticamente no login da tty1 (ambiente usa `zsh`, não `bash`)

### Uso
```bash
systemctl status miracle-wifid   # já deve estar rodando via systemd
export DISPLAY=:0
export XAUTHORITY=/root/.Xauthority
miracle-sinkctl
run <ID_DO_LINK>   # confirme o ID com "list" dentro do sinkctl
```
No dispositivo emissor: Windows (`Win+K` → "Conectar") ou Android (opção nativa de
transmitir tela).

### Limitações conhecidas
- Delay de ~1 a 2 segundos entre a ação no dispositivo do professor e a exibição no
  projetor, atribuído à decodificação H.264 por software concorrendo com a pilha
  Wi-Fi Direct num SoC de baixa capacidade (Cortex-A55 + Mali-G31 sem decodificador
  de vídeo dedicado ativo no caminho usado)
- Ocasionalmente queda de FPS ou falha total em exibir a imagem compartilhada
- Requer uma única interface Wi-Fi dedicada exclusivamente ao Wi-Fi Direct — a
  administração remota do dispositivo passa a depender de uma interface Ethernet

## Caminho 2 — Alternativa via GStreamer/UDP (sem Wi-Fi Direct)

Testado como comparação de latência e como alternativa multiplataforma (funciona em
qualquer sistema com GStreamer, incluindo Linux, contornando a incompatibilidade do
Miracast com determinados clientes Windows). Latência observada foi mínima.

**No dispositivo emissor** (ex.: Linux com X11):
```bash
gst-launch-1.0 ximagesrc ! videoconvert ! video/x-raw,format=I420 \
  ! x264enc tune=zerolatency speed-preset=ultrafast key-int-max=15 bitrate=4000 \
  ! mpegtsmux alignment=7 ! udpsink host=<IP_DA_TV_BOX> port=5000 buffer-size=60000
```

**Na TV Box (receptor)**:
```bash
export DISPLAY=:0
export XAUTHORITY=/root/.Xauthority
gst-launch-1.0 udpsrc port=5000 ! tsdemux ! h264parse ! avdec_h264 \
  ! videoconvert ! xvimagesink sync=false
```

## Agradecimentos
IFPR Campus Capanema, UTFPR Campus Francisco Beltrão, Receita Federal do Brasil, e ao
professor orientador Ederson Kobs.
