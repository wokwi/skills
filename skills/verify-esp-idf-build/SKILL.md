---
name: verify-esp-idf-build
description: Run and verify ESP-IDF firmware on the Wokwi simulator from idf.py (`idf.py wokwi`, the idf-wokwi extension), no board attached. Use to smoke-test an ESP-IDF 6.x build, check that it prints a serial message, or run it without a devkit, including right after an agent built it.
---

# Run ESP-IDF firmware on Wokwi from idf.py

`idf-wokwi` adds a `wokwi` action to `idf.py`: it reads the build directory (target chip,
`flasher_args.json`, ELF), generates a diagram with the matching devkit, uploads the firmware
and streams the serial console back. Needs ESP-IDF 6.0 or newer (`idf.py` extensions
arrived in 6.0; check `idf.py --version`) and idf-wokwi 1.2 or newer; on older ESP-IDF trees
use `wokwi-cli` instead.

1. **Install into the ESP-IDF Python environment, not the system one.**

   ```bash
   . $IDF_PATH/export.sh
   python -m pip install idf-wokwi
   idf.py wokwi --help          # the action must show up here
   ```

   If `idf.py` does not know `wokwi`, the package went into another interpreter.

2. **Token.** `export WOKWI_CLI_TOKEN=wok_...` from https://wokwi.com/dashboard/ci. Without
   it the command stops (exit 1) and prints that URL.

3. **Build and run.**

   ```bash
   idf.py set-target esp32s3
   idf.py wokwi    # builds first; no expect/fail/timeout flag = interactive, Ctrl+C ends
   ```

4. **Run it as a test.** Use this form in scripts and agent loops: it returns.

   ```bash
   idf.py wokwi --timeout 20000 --expect-text 'app: ready' --fail-text 'Guru Meditation Error'
   ```

   Exit 0 when the text appears, 1 on the fail text, 42 on timeout:

   ```text
   I (229) app: ready
   Expected text found: app: ready      # exit 0
   Fail text detected: app: ready       # exit 1 (same line given as --fail-text)
   Timeout after 3000 ms                # exit 42
   ```

   `--expect-regex` and `--fail-regex` match whole serial lines. Keep `--expect-text` on the
   last line of a successful run, so a pass means the whole program ran, not just the banner.

5. **Peripherals and Wi-Fi.** The generated diagram is the bare devkit wired to the serial
   monitor; a `diagram.json` in the project root (or `--diagram-file`) takes precedence.
   Wi-Fi works out of the box: the open access point `Wokwi-GUEST` has internet access, and
   a `wokwi-wifi-ap` part in `diagram.json` adds a network with your own SSID and password.
   Buttons and sensors are driven by `wokwi-cli` automation scenarios, not by `idf.py wokwi`.

## Pitfalls

- **A custom `diagram.json` must wire the console pins to `$serialMonitor`** (`TX0`/`RX0`
  on the S31 coreboard), or the firmware boots, nothing is read, and the run only times out.
- **`--timeout` is wall-clock milliseconds on your machine** (idf-wokwi 1.1.0 uses a
  monotonic clock), unlike `wokwi-cli`, where the same flag is simulated time. A slow link
  or busy server eats into it: give boot plus Wi-Fi plus NTP 20 to 60 s. `0` = no timeout.
- **On-premise server.** `WOKWI_CLI_SERVER=ws://host:port`; the same flags apply.

Docs: https://docs.wokwi.com/wokwi-ci/idf-wokwi-usage.
