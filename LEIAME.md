# 📷 Câmera Interna K1C 2025 — Correção 16:9 Real / 1280×720

🌐 **Idioma:** Português (Brasil) · [EN English](./README.md)

> [!WARNING]
> **Aviso**  
> Este procedimento **não foi validado nem endossado pelo C0DEbrained**. Trata-se de um contorno independente, documentado a partir de testes realizados em uma K1C 2025. Cada pessoa que optar por segui-lo deve executá-lo **em sua própria máquina, por sua própria conta e risco**. Faça backup de todos os arquivos que forem alterados e esteja preparado para restaurar a configuração original, se necessário.

## 🎯 Objetivo

Este procedimento corrige um problema observado na **Creality K1C 2025** em que o Mainsail está configurado com `aspect_ratio: 16:9`, mas a câmera continua aparecendo em **4:3**.

No equipamento testado, o problema não estava no Mainsail. A câmera suportava 1280×720 normalmente, porém o `mjpg_streamer` nativo da Creality abria `/dev/video0` em **640×480**, e sua versão de `input_uvc.so` falhava quando recebia o parâmetro `-r 1280x720`.

A solução validada foi usar uma versão antiga e compatível do **mjpg-streamer do Entware**, mantendo o firmware original intacto e fazendo o serviço da câmera usar os binários/plugins de `/opt`.

> [!NOTE]
> **Ambiente validado:** K1C 2025, arquitetura MIPS, câmera interna em `/dev/video0`, Mainsail, Helper Script 2025 de C0DEbrained.  
> **Resultado validado:** MJPEG **1280×720 @ 15 fps**, proporção **16:9 real**, persistente após reinicialização.

---

## 1. 🔎 Sintoma

Mesmo com a webcam configurada no Moonraker/Mainsail como:

```ini
[webcam chassis]
enabled: True
location: printer
service: mjpegstreamer
target_fps: 15
target_fps_idle: 5
stream_url: /webcam/?action=stream
snapshot_url: /webcam/?action=snapshot
flip_horizontal: False
flip_vertical: False
rotation: 0
aspect_ratio: 16:9
```

a imagem continua 4:3.

O motivo é que `aspect_ratio: 16:9` não muda a resolução capturada pela câmera; ele apenas descreve ao frontend como o stream deve ser tratado.

---

## 2. 🧪 Confirmar a causa

Entre por SSH na impressora e verifique a resolução atual:

```sh
v4l2-ctl -d /dev/video0 --get-fmt-video
```

No caso problemático, o resultado é:

```text
Width/Height : 640/480
Pixel Format : 'MJPG'
```

Isso é 4:3.

Agora confirme que a câmera realmente suporta 16:9:

```sh
v4l2-ctl -d /dev/video0 --list-formats-ext
```

Na K1C 2025 testada, `/dev/video0` oferece, entre outros modos:

```text
640x360
640x480
800x600
1280x720
1280x960
1920x1080
```

inclusive `1280x720` em MJPG a 15 fps.

---

## 3. ⚠️ Por que o streamer nativo não resolve

O serviço instalado pelo Helper Script 2025 inicia aproximadamente assim:

```sh
/usr/bin/mjpg_streamer -b \
  -i "/usr/lib/mjpg-streamer/input_uvc.so -d /dev/video0 -f 15" \
  -o "/usr/lib/mjpg-streamer/output_http.so -p 8080"
```

Como não há `-r`, o plugin abre a câmera em 640×480.

Adicionar `-r 1280x720` ao conjunto nativo causa, no equipamento testado:

```text
/usr/bin/mjpg_streamer: symbol lookup error:
/usr/lib/mjpg-streamer/input_uvc.so: undefined symbol: parse_resolution_opt
```

Esse erro é conhecido no `mjpg-streamer`: `parse_resolution_opt` fica em `utils.c`, e determinadas builds do plugin `input_uvc` não o vinculam corretamente.

Também não resolve configurar 1280×720 com `v4l2-ctl` antes de iniciar o streamer, pois o `input_uvc` nativo volta a configurar o dispositivo em 640×480 durante a inicialização.

---

## 4. 📦 Instalar o Entware

Se o Entware ainda não estiver instalado, abra o Helper Script:

```sh
sh /usr/data/helper-script/helper.sh
```

No menu, instale **Entware**.

Depois confirme:

```sh
ls -lh /opt/bin/opkg
ls -lh /opt/lib/ld.so.1
/opt/bin/opkg --version
```

O binário antigo do Entware usa `/opt/lib/ld.so.1`, portanto esse runtime precisa existir.

---

## 5. ⬇️ Baixar a versão compatível do mjpg-streamer

A versão validada neste procedimento foi:

- `mjpg-streamer` **2019-05-24-1**
- `mjpg-streamer-input-uvc` **2019-05-24-1**
- `mjpg-streamer-output-http` **2019-05-24-1**
- arquitetura Entware: **mipsel-3.4 / mipselsf-k3.4**
- dependência histórica: `libjpeg` **9c-2**

Use o `curl` fornecido pelo próprio Helper Script:

```sh
chmod +x /usr/data/helper-script/files/fixes/curl
CURL=/usr/data/helper-script/files/fixes/curl
cd /tmp
```

Baixe os pacotes:

```sh
$CURL -L \
http://bin.entware.net/mipselsf-k3.4/archive/mjpg-streamer_2019-05-24-1_mipsel-3.4.ipk \
-o /tmp/mjpg-streamer.ipk

$CURL -L \
http://bin.entware.net/mipselsf-k3.4/archive/mjpg-streamer-input-uvc_2019-05-24-1_mipsel-3.4.ipk \
-o /tmp/mjpg-streamer-input-uvc.ipk

$CURL -L \
http://bin.entware.net/mipselsf-k3.4/archive/mjpg-streamer-output-http_2019-05-24-1_mipsel-3.4.ipk \
-o /tmp/mjpg-streamer-output-http.ipk
```

Se o `opkg` reclamar da dependência `libjpeg`, baixe também:

```sh
$CURL -L \
http://bin.entware.net/mipselsf-k3.4/archive/libjpeg_9c-2_mipsel-3.4.ipk \
-o /tmp/libjpeg.ipk
```

Confirme que os downloads não são páginas HTML vazias:

```sh
ls -lh /tmp/*.ipk
file /tmp/mjpg-streamer.ipk
```

O pacote deve ser reconhecido como arquivo compactado, não como HTML/texto.

---

## 6. 🧩 Instalar os pacotes

Se `libjpeg` for necessário:

```sh
/opt/bin/opkg install /tmp/libjpeg.ipk
```

Depois:

```sh
/opt/bin/opkg install \
  /tmp/mjpg-streamer.ipk \
  /tmp/mjpg-streamer-input-uvc.ipk \
  /tmp/mjpg-streamer-output-http.ipk
```

Confirme:

```sh
ls -lh /opt/bin/mjpg_streamer
ls -lh /opt/lib/mjpg-streamer/input_uvc.so
ls -lh /opt/lib/mjpg-streamer/output_http.so
```

---

## 7. ✅ Testar 1280×720 antes de alterar o serviço

Pare qualquer streamer existente:

```sh
killall mjpg_streamer 2>/dev/null
```

Execute a versão do Entware em primeiro plano:

```sh
/opt/bin/mjpg_streamer \
  -i "/opt/lib/mjpg-streamer/input_uvc.so -d /dev/video0 -r 1280x720 -f 15" \
  -o "/opt/lib/mjpg-streamer/output_http.so -p 8080"
```

No equipamento validado, a saída mostra:

```text
MJPG Streamer Version.: 2.0
i: Using V4L2 device.: /dev/video0
i: Desired Resolution: 1280 x 720
i: Frames Per Second.: 15
i: Format............: JPEG
o: HTTP TCP port.....: 8080
```

Avisos do tipo abaixo podem aparecer:

```text
UVCIOC_CTRL_ADD - Error at Pan/Tilt/Focus/LED ... Inappropriate ioctl for device (25)
```

Eles indicam controles UVC opcionais não suportados pela câmera e, no teste realizado, **não impediram o stream**.

Abra o Mainsail e confirme visualmente o 16:9.

Em outro SSH, confirme:

```sh
v4l2-ctl -d /dev/video0 --get-fmt-video
```

Resultado esperado:

```text
Width/Height : 1280/720
Pixel Format : 'MJPG'
```

> [!TIP]
> Depois de confirmar o stream em 16:9, interrompa o processo em primeiro plano com `Ctrl+C` antes de aplicar a alteração permanente no serviço.

---

## 8. 💾 Tornar a correção permanente

Na instalação testada, o serviço ativo é:

```text
/etc/appetc/init.d/S57builtin_camera
```

Faça backup:

```sh
cp /etc/appetc/init.d/S57builtin_camera \
   /etc/appetc/init.d/S57builtin_camera.before-720p
```

Altere o serviço para usar o conjunto Entware e definir a resolução:

```sh
sed -i \
-e 's|/usr/bin/mjpg_streamer|/opt/bin/mjpg_streamer|g' \
-e 's|/usr/lib/mjpg-streamer/input_uvc.so|/opt/lib/mjpg-streamer/input_uvc.so|g' \
-e 's|/usr/lib/mjpg-streamer/output_http.so|/opt/lib/mjpg-streamer/output_http.so|g' \
-e 's|-d /dev/video0 -f 15|-d /dev/video0 -r 1280x720 -f 15|g' \
/etc/appetc/init.d/S57builtin_camera
```

Para evitar que uma reinstalação pelo Helper Script recoloque o arquivo antigo, aplique a mesma alteração à cópia do serviço no repositório local:

```sh
sed -i \
-e 's|/usr/bin/mjpg_streamer|/opt/bin/mjpg_streamer|g' \
-e 's|/usr/lib/mjpg-streamer/input_uvc.so|/opt/lib/mjpg-streamer/input_uvc.so|g' \
-e 's|/usr/lib/mjpg-streamer/output_http.so|/opt/lib/mjpg-streamer/output_http.so|g' \
-e 's|-d /dev/video0 -f 15|-d /dev/video0 -r 1280x720 -f 15|g' \
/usr/data/helper-script/files/services/S50builtin_camera-k1c-2025
```

Confira:

```sh
grep mjpg_streamer /etc/appetc/init.d/S57builtin_camera
```

A linha principal deve conter:

```text
/opt/bin/mjpg_streamer
/opt/lib/mjpg-streamer/input_uvc.so
-r 1280x720 -f 15
/opt/lib/mjpg-streamer/output_http.so
-p 8080
```

Reinicie o serviço:

```sh
/etc/appetc/init.d/S57builtin_camera stop
sleep 1
/etc/appetc/init.d/S57builtin_camera start
sleep 3
```

Valide:

```sh
ps w | grep '[m]jpg_streamer'
v4l2-ctl -d /dev/video0 --get-fmt-video
```

A resolução deve permanecer em `1280/720`.

Por fim, reinicie a impressora e confirme novamente:

```sh
reboot
```

Após o boot:

```sh
v4l2-ctl -d /dev/video0 --get-fmt-video
```

---

## 9. 🛠️ Diagnóstico rápido de problemas

### 🔸 Continua em 640×480

Verifique o processo real:

```sh
PID=$(pidof mjpg_streamer)
tr '\0' ' ' < /proc/$PID/cmdline
echo
```

Ele precisa apontar para `/opt/bin/mjpg_streamer`, carregar plugins de `/opt/lib/mjpg-streamer/` e conter `-r 1280x720`.

### 🔸 Aparece `undefined symbol: parse_resolution_opt`

Você provavelmente ainda está usando o plugin nativo:

```text
/usr/lib/mjpg-streamer/input_uvc.so
```

Use:

```text
/opt/lib/mjpg-streamer/input_uvc.so
```

junto com `/opt/bin/mjpg_streamer`.

### 🔸 `/opt/bin/mjpg_streamer: not found`

O Entware ou os pacotes antigos ainda não estão instalados corretamente.

### 🔸 O vídeo não abre na porta 8080

Confira:

```sh
ps w | grep '[m]jpg_streamer'
netstat -lntp 2>/dev/null | grep ':8080'
```

### 🔸 O Mainsail ainda reserva proporção errada

Confirme que o bloco da câmera contém:

```ini
aspect_ratio: 16:9
```

Depois reinicie Moonraker/Mainsail conforme necessário.

---

## 10. ↩️ Rollback

Para voltar ao serviço anterior:

```sh
killall mjpg_streamer 2>/dev/null

cp /etc/appetc/init.d/S57builtin_camera.before-720p \
   /etc/appetc/init.d/S57builtin_camera

chmod 755 /etc/appetc/init.d/S57builtin_camera
/etc/appetc/init.d/S57builtin_camera start
```

Não remova o Entware se outros recursos da impressora dependem dele.

---

## 🧠 Causa raiz resumida

```text
K1C 2025 camera (/dev/video0)
        │
        ├── suporta MJPG 1280×720 @ 15 fps  ✅
        │
        └── mjpg_streamer nativo
                │
                ├── sem -r  → abre 640×480   ❌ 4:3
                │
                └── com -r  → parse_resolution_opt ausente ❌

Entware mjpg-streamer 2019-05-24-1
        │
        └── input_uvc aceita -r 1280x720
                │
                └── MJPEG 1280×720 @ 15 fps ✅ 16:9
```

## 📚 Referências técnicas

- C0DEbrained/Creality-Helper-Script-2025 — serviço atual da câmera K1C 2025: https://github.com/C0DEbrained/Creality-Helper-Script-2025/blob/main/files/services/S50builtin_camera-k1c-2025
- C0DEbrained/Creality-Helper-Script-2025 — lógica de instalação/configuração da câmera: https://github.com/C0DEbrained/Creality-Helper-Script-2025/blob/main/scripts/usb_camera.sh
- mjpg-streamer upstream — problema `parse_resolution_opt`: https://github.com/jacksonliam/mjpg-streamer/issues/414
- Registro histórico dos pacotes Entware 2019-05-24-1 e `libjpeg 9c-2`: https://forum.keenetic.ru/topic/7713-mjpg-streamer-%D0%BF%D0%BE%D0%B4%D0%BA%D0%BB%D1%8E%D1%87%D0%B5%D0%BD%D0%B8%D0%B5-%D0%B2%D0%B5%D0%B1-%D0%BA%D0%B0%D0%BC%D0%B5%D1%80%D1%8B/page/3/
