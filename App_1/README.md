# App 1 scaffold — setup, blink, web monitor

This scaffold is **~80% complete**. It compiles and runs as-is. Your job is theme customization + the README defense.

## What's already done for you

- Dual-core architecture: `blink_task` on Core 1, HTTP server on Core 0
- `vTaskDelayUntil` for drift-free 1 Hz blinking
- Wi-Fi station mode wired to Wokwi's virtual AP
- HTTP server returns themed HTML with auto-refresh
- Wokwi diagram: ESP32-S3 + green LED on GPIO 2 + 1 kΩ current-limit resistor
- ESP-IDF v5.x project skeleton with sdkconfig defaults

## What you do

### 1. Theme rename pass

Search the project for `YOURTHEME` and replace every occurrence with your chosen theme name. There are ~6 occurrences in `main.c`.

```bash
# Linux / macOS
grep -rn YOURTHEME .
# Or in your editor: project-wide find/replace
```

### 2. Customize the web page

In `main.c`, find `handle_root()`. The HTML there is generic. Make it theme-specific:

| Theme | Example |
|-------|---------|
| Avionics | `<h1>UAV-01 attitude beacon</h1>` + tail-number, status, last-heartbeat |
| Medical | `<h1>Pulse monitor — Bed 4-A</h1>` + BPM, last-pulse-time, alarm state |
| Space | `<h1>SAT-1 health beacon</h1>` + mission elapsed time, downlink ready |
| Industrial | `<h1>Ride-X dispatch readiness</h1>` + ready/locked status |
| Security | `<h1>Enclave-A integrity beacon</h1>` + attested/tampered |

### 3. Optionally adjust parameters

- `BLINK_PERIOD_MS` — try faster (250 ms) or slower (2 s)
- `LED_GPIO` — if your hardware uses a different pin

### 4. Verify and document

Run on Wokwi (or hardware). Open the IoT Gateway, point your browser at the device's IP. Confirm:

- LED toggles at the expected rate
- Web page shows ON/OFF and auto-refreshes
- Serial log shows `[YOURTHEME] beacon = ON/OFF` messages

Screenshot the web page. Put it in your README.

## README content required for submission

1. **Theme banner** — which theme, why you picked it (1–2 sentences)
2. **System summary** — paragraph describing what this app does in YOUR theme
3. **Wokwi link** — public URL to your project
4. **Concurrency diagram** — boxes (tasks), arrows (shared state), Core assignments
5. **Engineering analysis answers** — see the assignment page
6. **AI disclosure** — every chat / URL / snippet used during development

## Setup in Wokwi

This scaffold lays out files for a local ESP-IDF build (`main/main.c`). A fresh Wokwi ESP-IDF template uses `main/src/main.c` instead &mdash; both layouts build, but `main/CMakeLists.txt` must match wherever `main.c` actually lives.

**Two-minute setup in a fresh Wokwi ESP-IDF project (`https://wokwi.com/projects/new/esp32-s3`):**

1. Replace these files in the Wokwi editor with the versions from this folder:
   - `diagram.json`
   - `wokwi.toml`
   - `sdkconfig.defaults` (create if Wokwi didn't generate one)
   - `main/CMakeLists.txt`
2. Place this folder's `main.c` at `main/main.c`. If Wokwi created `main/src/main.c`, either:
   - **(easier)** Delete `main/src/` and put `main.c` directly under `main/`, OR
   - **(no file moves)** Leave `main/src/main.c` and edit `main/CMakeLists.txt` so `SRCS "main.c"` becomes `SRCS "src/main.c"` and `INCLUDE_DIRS "."` becomes `INCLUDE_DIRS "src"`.
3. Confirm the top-level `CMakeLists.txt` `project(...)` name matches the `firmware` / `elf` paths in `wokwi.toml`. If you see "firmware not found," that's the mismatch.
4. Click &#9654; in the Wokwi toolbar. First build is ~30&ndash;60 s; subsequent builds use ccache and finish in seconds.

### Build locally with ESP-IDF instead

```bash
# Set up ESP-IDF (one-time)
. $HOME/esp/esp-idf/export.sh

idf.py set-target esp32s3
idf.py build
idf.py -p /dev/ttyUSB0 flash monitor
```

## Viewing the web page

Once Wi-Fi connects, the serial monitor prints a line like:

```
I (4582) app1: Got IP: 10.13.37.2
```

In Wokwi, when an HTTP server starts listening on port 80, a small **globe / network indicator** appears in the simulator panel. Click it and choose "Open in new tab" &mdash; the served HTML opens in your browser. The page polls a `/state` JSON endpoint at 4 Hz via JavaScript (no full-page reload), so you'll see the ON/OFF indicator and the toggle counter update smoothly in time with the LED blink &mdash; no Nyquist aliasing against the 1 Hz toggle rate.

If no indicator appears, the serial log alone is sufficient proof of life &mdash; every `[YOURTHEME] beacon = ON/OFF` line means the blink task completed an iteration. The web view is the friendlier UX, not the only evidence.

For local ESP-IDF builds, the IP printed in the serial log is reachable on your LAN: open `http://<that-ip>/` in any browser on the same network.

## Engineering analysis prompts (answer in your README)

1. **Why two tasks?** What's the failure mode of a single super-loop that polls the web server AND blinks the LED?
2. **Why pin to specific cores?** What problem does pinning solve that round-robin scheduling doesn't?
3. **Why does a single `bool` (our `led_on`) not need a mutex here, but a `struct { bool; uint32_t; }` would?** Reference what you know about atomic reads on Xtensa.

## Common pitfalls

- **Page won't load.** Check the serial monitor for the `Got IP` line. If you have it but the page won't open, the network indicator in the simulator panel is the simplest path; Wokwi Pro's IoT Gateway is a separate (paid) option that publishes the device to the open internet.
- **LED doesn't blink in Wokwi.** Confirm the diagram.json has the LED on GPIO 2 with a 1 kΩ resistor to GND.
- **Build fails with "esp_http_server.h not found."** Your ESP-IDF version is too old. Need v5.0+.
- **HTTP server crashes when refreshing fast.** That's a teachable moment for App 5 (IPC + back-pressure). For App 1, slow your refresh down.

## Honor code reminder

Use AI freely. Document its use. You will be asked to explain any line you submit.
