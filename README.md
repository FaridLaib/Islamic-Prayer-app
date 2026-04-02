# ESP32-S3 Prayer Clock

A smart desk clock for an 800×480 RGB display, built on an ESP32-S3 with 16MB flash. It shows the current time, Islamic prayer times, live weather, and plays an adhan (call to prayer) through a MAX98357A I2S amplifier — all behind a custom full-screen background image loaded from flash.

Exact model: https://github.com/VIEWESMART/UEDX80480070ESP32-7inch-Touch-Display

Purchase from here: (not affiliated)
https://www.aliexpress.us/item/3256807528358359.html?spm=a2g0o.order_list.order_list_main.10.5da81802e2ZPok&gatewayAdapt=glo2usa

---

## Features

- **Prayer times** — Fajr, Sunrise, Dhuhr, Asr, Maghrib, Isha fetched over Wi-Fi and highlighted as the next prayer approaches
- **Live weather** — current conditions + 7-day forecast pulled from weather.gov (US), shown on a swipe-left screen
- **Adhan playback** — plays an MP3 adhan at Maghrib (and a configurable alarm) via an I2S amplifier
- **Chronograph** — tap-to-start stopwatch screen, accessible by swipe
- **Captive-portal setup** — on first boot, the device creates a `PrayerClock-Setup` Wi-Fi hotspot; connect and visit `192.168.4.1` to enter your Wi-Fi credentials, city, and timezone — no USB re-flashing needed
- **Custom background** — a 800×480 RGB565 image stored in LittleFS (PSRAM-backed), swappable without recompiling the firmware
- **NVS persistence** — all settings survive power cycles

---

## Hardware

| Part | Notes |
|---|---|
| ESP32-S3 dev board | 16 MB flash, PSRAM required |
| 800×480 RGB LCD | Parallel RGB interface |
| MAX98357A | I2S mono amplifier; BCLK → GPIO 10, LRC → GPIO 13, DIN → GPIO 11 |
| Speaker | 4–8 Ω |

---

## Flashing the Firmware

The source code is not published. A pre-built `.bin` is provided instead.

### What you need

- [esptool.py](https://github.com/espressif/esptool) — `pip install esptool`
- Or the standalone `esptool.exe` bundled with Arduino (found at `%APPDATA%\Local\Arduino15\packages\esp32\tools\esptool_py\`)

### Flash command

Open a terminal in the folder containing the `.bin` file and run:

```bash
esptool.py --chip esp32s3 \
           --port COM9 \
           --baud 460800 \
           --before default-reset \
           --after hard-reset \
           write-flash 0x10000 LVGL_Arduino.bin
```

> **Windows:** replace `COM9` with your actual COM port (check Device Manager).  
> **Linux/macOS:** replace `COM9` with `/dev/ttyUSB0` or `/dev/tty.usbserial-*`.

After flashing, power-cycle the board. If this is the first time, the display will show a Wi-Fi setup screen — follow the on-screen instructions.

---

## Uploading Custom Files (Background Image & Audio)

The device uses a **LittleFS partition** at flash offset `0x310000` (9.9 MB) to store:

| File | Purpose |
|---|---|
| `data/background.bin` | Full-screen background image (RGB565, 800×480) |
| `data/chrono_alarm.mp3` | Stopwatch alarm sound |
| `data/maghrib_adhan.mp3` | Adhan played at Maghrib |

You upload these files independently of the firmware — no re-flashing the app needed.

---

### Step 1 — Prepare your `data/` folder

Create a folder called `data` and place your files inside it. At minimum, `background.bin` must be present.

---

### Step 2 — Convert a background image to `background.bin`

The background must be a **800×480 PNG**, converted to raw RGB565 binary format using the LVGL image converter.

**Option A — LVGL online converter (easiest)**

1. Go to [LVGL Image Converter](https://lvgl.io/tools/imageconverter)
2. Upload your 800×480 PNG
3. Set **Color format** → `RGB565`
4. Set **Output format** → `Binary`
5. Download the result and rename it `background.bin`
6. Place it in your `data/` folder

**Option B — Python script**

If you have the `lv_img_conv` Python tool:

```bash
python Converter.py --cf RGB565 --ofmt BIN --name background -o ./data Background.png
```

---

### Step 3 — Upload with `upload_littlefs.bat` (Windows)

A batch script is included that automates building and flashing the LittleFS image.

**Before running**, open `upload_littlefs.bat` and edit the top section to match your setup:

```bat
SET COM_PORT=COM9             ← your ESP32's COM port
SET SKETCH_DIR=C:\Arduino\...\LVGL_Arduino   ← path to the folder containing data\
```

The script expects the Arduino ESP32 core tools at their default locations. If you installed Arduino IDE to a non-standard path, also update the `copy` lines near the top.

**Then just double-click `upload_littlefs.bat`.**

It will:
1. Copy `mklittlefs.exe` and `esptool.exe` to `C:\esp32tools\` (avoids path-with-spaces issues)
2. Pack your `data/` folder into a LittleFS image
3. Prompt you to close Arduino IDE (so the COM port is free)
4. Flash the image to offset `0x310000` at 460800 baud

After the upload completes, power-cycle the board and watch Serial Monitor for:

```
bg: loaded OK
```

---

### Step 3 (alternative) — Manual upload (any OS)

If you're on Linux/macOS or prefer the command line:

```bash
# 1. Build the LittleFS image
mklittlefs -c ./data -s 0x9F0000 -p 256 -b 4096 littlefs.bin

# 2. Flash it
esptool.py --chip esp32s3 \
           --port /dev/ttyUSB0 \
           --baud 460800 \
           write-flash 0x310000 littlefs.bin
```

`mklittlefs` can be downloaded from the [ESP32 LittleFS Uploader releases](https://github.com/FaridLaib/ESP32-LittleFS-uploader) or extracted from your Arduino15 packages folder.

---

### Partition reference

| Partition | Offset | Size |
|---|---|---|
| App (firmware) | `0x10000` | 3 MB |
| LittleFS (data) | `0x310000` | 9.9 MB |

These values match the custom `partitions.csv` the firmware was compiled with. Do **not** change them.

---

## Replacing the Adhan / Alarm Sounds

Drop replacement MP3 files into your `data/` folder before running `upload_littlefs.bat`:

| Filename | Triggered by |
|---|---|
| `maghrib_adhan.mp3` | Automatically at Maghrib prayer time |
| `chrono_alarm.mp3` | Stopwatch alarm |

Any standard mono or stereo MP3 works. Shorter files (under ~2 MB) load faster from LittleFS.

---

## First-Time Wi-Fi Setup

1. Power on the board with no saved credentials
2. The display shows: **"Connect to Wi-Fi: PrayerClock-Setup"**
3. On your phone, join that network
4. Open a browser and navigate to `192.168.4.1`
5. Enter your home Wi-Fi SSID, password, city name, and timezone
6. Tap **Save & Connect** — the device restarts and connects automatically

Settings are stored in NVS and survive future firmware updates (as long as the NVS partition is not erased).


![Display](images/IMG_3950.PNG)

