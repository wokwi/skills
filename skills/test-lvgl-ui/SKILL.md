---
name: test-lvgl-ui
description: Test an LVGL (or any other) embedded GUI in CI on the Wokwi simulator. Scripted taps and button presses drive the screen; the simulated display is compared pixel by pixel against reference PNGs, without a board. Use to test or regression-check a display UI, screen, menu or widget on ESP32 or other MCUs, or when someone mentions LVGL, TFT_eSPI, ILI9341, ST7789, SSD1306, a touchscreen, `take-screenshot` or `compare-with`.
---

# Test an LVGL UI in CI

The simulator draws the display from the firmware's own pixel writes; the same build and
scenario give the same PNG, byte for byte. Needs wokwi-cli 0.29 or newer, the display part
in `diagram.json` with a stable id (`lcd1`; a built-in panel uses the board id) and
`wokwi.toml` pointing at the firmware and ELF.

1. **Make the firmware narrate screen changes:** one line when a screen is ready, one per
   state change (`Tick: 1`, `stroke at 60,100`). A silent UI needs guessed delays.

2. **Capture the references once** with `save-to`, look at the PNGs, commit them:

   ```yaml
   name: hello-s31 boot + counter
   version: 1
   steps:
     - wait-serial: "Tick: 1"    # label reads "Tick: 0" and has been on screen ~1 s
     - take-screenshot: { part-id: lcd1, save-to: screenshots/boot.png }
     - wait-serial: "Tick: 6"
     - take-screenshot: { part-id: lcd1, save-to: screenshots/tick-5.png }
   ```

   ```bash
   wokwi-cli . --scenario test.yaml --timeout 20000
   # [hello-s31 boot + counter] Scenario completed successfully   (exit 0)
   ```

3. **Add `compare-with: screenshots/boot.png`** next to each `save-to`, and point `save-to`
   at `out/boot.png` so a red run keeps the actual image. A mismatch stops the run at the
   first differing pixel (here the boot frame was checked against the tick-5 reference):

   ```text
   Error: [2287494385ns] Screenshot mismatch for screenshots/tick-5.png at x,y=(88,206)
   ```

   Exit 1 on mismatch, missing reference or wrong size (`has a different size (120x160)
   than the part's screen (240x320)`); exit 42 when `--timeout` (simulated ms) runs out first.

4. **Drive the UI.** `touch` taps (released after 50 ms), `touch-press` / `touch-move` /
   `touch-release` drag, `set-control` presses buttons. Wait for the firmware's line, then
   screenshot as in step 3:

   ```yaml
     - touch-press: { part-id: esp, x: 100, y: 150 }
     - wait-serial: 'stroke at 100,150'
     - touch-move: { part-id: esp, x: 240, y: 270 }
     - delay: 50ms
     - touch-release: { part-id: esp }
   ```

5. **In CI** run `wokwi/wokwi-ci-action@v1` with `scenario: test.yaml` and `timeout: 20000`
   (default 10 s), then upload `out/` with `if: always()`.

## Pitfalls

- **Pixel-exact.** A font, an LVGL or driver version, colour depth, rotation or a build
  flag changes pixels. Regenerate the references in the same commit as the UI change.
- **Paths are relative to the scenario file** (both keys), not to the project root or cwd.
- **Capture one event late.** LVGL redraws on its own timer; to capture `Tick: 0`, wait
  for `Tick: 1` (a full period has passed, as in step 2).
- **Touch coordinates are the touch controller's**, not the display's: on ILI9341 + FT6206
  the origin is bottom-right and the firmware maps it. Print what the firmware sees.
- **A `wait-serial` sees only output after it starts.** Put it right after the action
  that triggers the line; a `delay` in between can swallow it.

Docs: https://docs.wokwi.com/wokwi-ci/automation-scenarios.
Demo: https://github.com/wokwi/esp32-s31-hello-wokwi-idf.
