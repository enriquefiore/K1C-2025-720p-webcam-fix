# 📷 K1C 2025 Built-in Camera — True 16:9 / 1280×720 Fix

🌐 **Language:** English · [PT-BR Português](./LEIAME.md)

> [!WARNING]
> **Disclaimer**  
> This procedure has **not been validated or endorsed by C0DEbrained**. It is an independent workaround documented from testing on a K1C 2025. Anyone choosing to follow it does so **on their own machine and at their own risk**. Make backups of any files you modify and be prepared to restore the original configuration if necessary.

## 🎯 Goal

This procedure fixes an issue observed on the **Creality K1C 2025** where Mainsail is configured with `aspect_ratio: 16:9`, but the camera is still displayed as **4:3**.

On the tested printer, Mainsail was not the root cause. The camera supported 1280×720 correctly, but Creality's native `mjpg_streamer` opened `/dev/video0` at **640×480**, and its `input_uvc.so` build failed whenever `-r 1280x720` was supplied.

The validated workaround is to use an older compatible **Entware mjpg-streamer** build, leave the original firmware files untouched, and make the camera service use the `/opt` binary and plugins.

> [!NOTE]
> **Validated environment:** K1C 2025, MIPS architecture, built-in camera on `/dev/video0`, Mainsail, C0DEbrained's Helper Script 2025.  
> **Validated result:** MJPEG **1280×720 @ 15 fps**, true **16:9**, persistent after reboot.

---

## 1. 🔎 Symptom

Even when Moonraker/Mainsail contains:

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

the image is still 4:3.

`aspect_ratio: 16:9` does not change the camera capture resolution; it only tells the frontend how the stream should be presented.

---

## 2. 🧪 Confirm the root cause

SSH into the printer and check the active capture format:

```sh
v4l2-ctl -d /dev/video0 --get-fmt-video
```

On the affected setup:

```text
Width/Height : 640/480
Pixel Format : 'MJPG'
```

That is 4:3.

Now verify that the camera itself supports widescreen modes:

```sh
v4l2-ctl -d /dev/video0 --list-formats-ext
```

On the tested K1C 2025, `/dev/video0` advertises modes including:

```text
640x360
640x480
800x600
1280x720
1280x960
1920x1080
```

including MJPG 1280×720 at 15 fps.

---

## 3. ⚠️ Why the native streamer does not solve it

The Helper Script 2025 camera service currently starts roughly as follows:

```sh
/usr/bin/mjpg_streamer -b \
  -i "/usr/lib/mjpg-streamer/input_uvc.so -d /dev/video0 -f 15" \
  -o "/usr/lib/mjpg-streamer/output_http.so -p 8080"
```

Since no resolution is supplied, this build opens the device at 640×480.

Adding `-r 1280x720` to the native binary/plugin pair causes the tested unit to fail with:

```text
/usr/bin/mjpg_streamer: symbol lookup error:
/usr/lib/mjpg-streamer/input_uvc.so: undefined symbol: parse_resolution_opt
```

This is a known mjpg-streamer build issue: `parse_resolution_opt` is implemented in `utils.c`, and some builds of the UVC plugin do not link it correctly.

Pre-setting the format with `v4l2-ctl` does not work either: the native UVC plugin reconfigures the camera back to 640×480 when it starts.

---

## 4. 📦 Install Entware

If Entware is not installed yet, start the Helper Script:

```sh
sh /usr/data/helper-script/helper.sh
```

Install **Entware** from the menu.

Then verify:

```sh
ls -lh /opt/bin/opkg
ls -lh /opt/lib/ld.so.1
/opt/bin/opkg --version
```

The legacy Entware binary expects `/opt/lib/ld.so.1`, so the Entware runtime must be present.

---

## 5. ⬇️ Download the compatible mjpg-streamer packages

The versions validated in this workaround are:

- `mjpg-streamer` **2019-05-24-1**
- `mjpg-streamer-input-uvc` **2019-05-24-1**
- `mjpg-streamer-output-http` **2019-05-24-1**
- Entware architecture: **mipsel-3.4 / mipselsf-k3.4**
- historical dependency: `libjpeg` **9c-2**

Use the curl binary shipped with the Helper Script:

```sh
chmod +x /usr/data/helper-script/files/fixes/curl
CURL=/usr/data/helper-script/files/fixes/curl
cd /tmp
```

Download the packages:

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

If `opkg` reports a missing `libjpeg` dependency, also download:

```sh
$CURL -L \
http://bin.entware.net/mipselsf-k3.4/archive/libjpeg_9c-2_mipsel-3.4.ipk \
-o /tmp/libjpeg.ipk
```

Verify that the downloads are real package files rather than an HTML error page:

```sh
ls -lh /tmp/*.ipk
file /tmp/mjpg-streamer.ipk
```

---

## 6. 🧩 Install the packages

If `libjpeg` is required:

```sh
/opt/bin/opkg install /tmp/libjpeg.ipk
```

Then install the streamer and plugins:

```sh
/opt/bin/opkg install \
  /tmp/mjpg-streamer.ipk \
  /tmp/mjpg-streamer-input-uvc.ipk \
  /tmp/mjpg-streamer-output-http.ipk
```

Verify:

```sh
ls -lh /opt/bin/mjpg_streamer
ls -lh /opt/lib/mjpg-streamer/input_uvc.so
ls -lh /opt/lib/mjpg-streamer/output_http.so
```

---

## 7. ✅ Test 1280×720 before changing the boot service

Stop any running streamer:

```sh
killall mjpg_streamer 2>/dev/null
```

Run the Entware build in the foreground:

```sh
/opt/bin/mjpg_streamer \
  -i "/opt/lib/mjpg-streamer/input_uvc.so -d /dev/video0 -r 1280x720 -f 15" \
  -o "/opt/lib/mjpg-streamer/output_http.so -p 8080"
```

On the validated printer the output includes:

```text
MJPG Streamer Version.: 2.0
i: Using V4L2 device.: /dev/video0
i: Desired Resolution: 1280 x 720
i: Frames Per Second.: 15
i: Format............: JPEG
o: HTTP TCP port.....: 8080
```

You may see warnings such as:

```text
UVCIOC_CTRL_ADD - Error at Pan/Tilt/Focus/LED ... Inappropriate ioctl for device (25)
```

These are unsupported optional UVC controls and did **not** prevent streaming on the tested camera.

Open Mainsail and confirm that the image is now 16:9.

From another SSH session, verify:

```sh
v4l2-ctl -d /dev/video0 --get-fmt-video
```

Expected result:

```text
Width/Height : 1280/720
Pixel Format : 'MJPG'
```

> [!TIP]
> Once the 16:9 stream is confirmed, stop the foreground process with `Ctrl+C` before applying the persistent service change.

---

## 8. 💾 Make the fix persistent

On the tested installation, the active service is:

```text
/etc/appetc/init.d/S57builtin_camera
```

Create a backup:

```sh
cp /etc/appetc/init.d/S57builtin_camera \
   /etc/appetc/init.d/S57builtin_camera.before-720p
```

Change the service to use the Entware build and explicitly request 1280×720:

```sh
sed -i \
-e 's|/usr/bin/mjpg_streamer|/opt/bin/mjpg_streamer|g' \
-e 's|/usr/lib/mjpg-streamer/input_uvc.so|/opt/lib/mjpg-streamer/input_uvc.so|g' \
-e 's|/usr/lib/mjpg-streamer/output_http.so|/opt/lib/mjpg-streamer/output_http.so|g' \
-e 's|-d /dev/video0 -f 15|-d /dev/video0 -r 1280x720 -f 15|g' \
/etc/appetc/init.d/S57builtin_camera
```

Also update the Helper Script's local service template so a reinstall does not immediately restore the old command:

```sh
sed -i \
-e 's|/usr/bin/mjpg_streamer|/opt/bin/mjpg_streamer|g' \
-e 's|/usr/lib/mjpg-streamer/input_uvc.so|/opt/lib/mjpg-streamer/input_uvc.so|g' \
-e 's|/usr/lib/mjpg-streamer/output_http.so|/opt/lib/mjpg-streamer/output_http.so|g' \
-e 's|-d /dev/video0 -f 15|-d /dev/video0 -r 1280x720 -f 15|g' \
/usr/data/helper-script/files/services/S50builtin_camera-k1c-2025
```

Verify:

```sh
grep mjpg_streamer /etc/appetc/init.d/S57builtin_camera
```

The main command should now contain:

```text
/opt/bin/mjpg_streamer
/opt/lib/mjpg-streamer/input_uvc.so
-r 1280x720 -f 15
/opt/lib/mjpg-streamer/output_http.so
-p 8080
```

Restart the camera service:

```sh
/etc/appetc/init.d/S57builtin_camera stop
sleep 1
/etc/appetc/init.d/S57builtin_camera start
sleep 3
```

Validate:

```sh
ps w | grep '[m]jpg_streamer'
v4l2-ctl -d /dev/video0 --get-fmt-video
```

The resolution should remain `1280/720`.

Finally reboot and confirm again:

```sh
reboot
```

After boot:

```sh
v4l2-ctl -d /dev/video0 --get-fmt-video
```

---

## 9. ⚠️ Known issue: Klipper may stop while a large timelapse is rendering

On the tested K1C 2025, a second issue was observed after **long/large prints** with Moonraker Timelapse enabled. When the print finishes and the timelapse starts rendering, the printer may appear temporarily frozen or become inaccessible from the web interface. When Mainsail becomes reachable again, it may report that **Moonraker is running but cannot connect to Klipper**, while the timelapse render is still in progress.

![Mainsail showing Moonraker disconnected from Klipper while timelapse is rendering](./assets/k1c_timelapse_klipper_disconnected.png)

In the observed case, the **Klipper service was no longer running** and had to be started again manually.

> [!IMPORTANT]
> This behavior has been **observed**, but its exact cause has not yet been confirmed. It should not be described as a proven consequence of the 1280×720 camera workaround. A likely area to investigate is resource pressure during FFmpeg/timelapse rendering on the K1C's limited-memory host, but logs are required to distinguish an OOM kill from a service/firmware interaction.

### Safe recovery after the print has fully finished

If the print is complete and Mainsail reports Klipper as disconnected, restart the Klipper service over SSH:

```sh
/etc/init.d/S55klipper_service restart
```

The Helper Script also exposes **Tools → Restart Klipper service**.

Before restarting Klipper, make sure the print has actually completed and there is no active motion that must be preserved.

### Capture diagnostics before restarting Klipper

If SSH is still available, collect the following information first:

```sh
date
uptime
free -m
ps w | grep -E '[k]lippy|[m]oonraker|[f]fmpeg|[m]jpg_streamer'
dmesg | grep -iE 'oom|out of memory|killed process' | tail -30
```

Also preserve the current logs when possible:

```sh
tail -n 200 /usr/data/printer_data/logs/klippy.log
tail -n 200 /usr/data/printer_data/logs/moonraker.log
```

If `dmesg` reports an OOM kill involving Klipper or another critical process, the correct fix is to reduce rendering pressure rather than simply auto-restarting Klipper.

### Recommended mitigation while this is being investigated

Moonraker Timelapse supports disabling automatic rendering at the end of a print. For large prints, consider setting:

```ini
autorender: False
```

This lets the print finish without immediately starting FFmpeg. Render the timelapse later, when a temporary Klipper interruption is less disruptive.

If automatic rendering is required and this issue persists, another conservative option is to use **640×360 @ 15 fps** for the camera. It remains true 16:9 but uses only one quarter of the pixels of 1280×720 per frame.

> [!CAUTION]
> Do **not** add an automatic Klipper restart loop yet. If Klipper is being killed because the system is under memory pressure, a watchdog may simply restart it into the same resource-starved condition. First identify whether the service is being killed by the kernel, exiting on its own, or being stopped by another process.

---

## 10. 🛠️ Quick troubleshooting

### 🔸 It is still 640×480

Inspect the real process command line:

```sh
PID=$(pidof mjpg_streamer)
tr '\0' ' ' < /proc/$PID/cmdline
echo
```

It must use `/opt/bin/mjpg_streamer`, load plugins from `/opt/lib/mjpg-streamer/`, and contain `-r 1280x720`.

### 🔸 `undefined symbol: parse_resolution_opt`

You are most likely still loading the native plugin:

```text
/usr/lib/mjpg-streamer/input_uvc.so
```

Use:

```text
/opt/lib/mjpg-streamer/input_uvc.so
```

with `/opt/bin/mjpg_streamer`.

### 🔸 `/opt/bin/mjpg_streamer: not found`

Entware or the archived packages have not been installed correctly.

### 🔸 Nothing is listening on port 8080

Check:

```sh
ps w | grep '[m]jpg_streamer'
netstat -lntp 2>/dev/null | grep ':8080'
```

### 🔸 Mainsail still reserves the wrong aspect ratio

Verify that the webcam block contains:

```ini
aspect_ratio: 16:9
```

Restart Moonraker/Mainsail as needed.

---

## 11. ↩️ Rollback

To restore the previous service:

```sh
killall mjpg_streamer 2>/dev/null

cp /etc/appetc/init.d/S57builtin_camera.before-720p \
   /etc/appetc/init.d/S57builtin_camera

chmod 755 /etc/appetc/init.d/S57builtin_camera
/etc/appetc/init.d/S57builtin_camera start
```

Do not remove Entware if other printer features depend on it.

---

## 🧠 Root cause summary

```text
K1C 2025 camera (/dev/video0)
        │
        ├── supports MJPG 1280×720 @ 15 fps ✅
        │
        └── native mjpg_streamer
                │
                ├── no -r  → opens 640×480   ❌ 4:3
                │
                └── with -r → missing parse_resolution_opt ❌

Entware mjpg-streamer 2019-05-24-1
        │
        └── input_uvc accepts -r 1280x720
                │
                └── MJPEG 1280×720 @ 15 fps ✅ 16:9
```

## 📚 Technical references

- C0DEbrained/Creality-Helper-Script-2025 — current K1C 2025 built-in camera service: https://github.com/C0DEbrained/Creality-Helper-Script-2025/blob/main/files/services/S50builtin_camera-k1c-2025
- C0DEbrained/Creality-Helper-Script-2025 — camera install/configuration logic: https://github.com/C0DEbrained/Creality-Helper-Script-2025/blob/main/scripts/usb_camera.sh
- mjpg-streamer upstream — `parse_resolution_opt` issue: https://github.com/jacksonliam/mjpg-streamer/issues/414
- Historical Entware package record for 2019-05-24-1 and `libjpeg 9c-2`: https://forum.keenetic.ru/topic/7713-mjpg-streamer-%D0%BF%D0%BE%D0%B4%D0%BA%D0%BB%D1%8E%D1%87%D0%B5%D0%BD%D0%B8%D0%B5-%D0%B2%D0%B5%D0%B1-%D0%BA%D0%B0%D0%BC%D0%B5%D1%80%D1%8B/page/3/
---
- Moonraker Timelapse — `autorender` configuration and render behavior: https://github.com/mainsail-crew/moonraker-timelapse/blob/main/docs/configuration.md
- Creality Helper Script — K1 tools menu includes a Klipper service restart action: https://github.com/Guilouz/Creality-Helper-Script/blob/main/scripts/menu/K1/tools_menu_K1.sh
