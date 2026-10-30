---
name: debug-esp32-firmware
description: Debug ESP32 firmware on the Wokwi simulator, no board or JTAG needed. Decode a panic backtrace, reproduce the bug with scripted inputs, capture I2C/SPI/UART traffic to a VCD file, attach GDB from the CLI or VS Code. Use for crashes, panics, watchdog resets, hangs, intermittent faults and wrong bus traffic.
---

# Debug ESP32 firmware on Wokwi

The simulator runs the real firmware with every pin visible and can pause it, so the same
inputs give the same crash at the same instruction. Needs wokwi-cli 0.29 or newer and the
ESP-IDF toolchain on `PATH` (`. $IDF_PATH/export.sh`).

1. **Decode the panic** with the ELF of the exact build that crashed (a rebuild moves every
   address). Exceptions print `Guru Meditation Error: Core  0 panic'ed (LoadProhibited)`,
   `abort()`/`ESP_ERROR_CHECK` print `abort() was called at PC 0x...`, `assert` prints
   `assert failed:`; on Xtensa all end in `Backtrace:` with `pc:sp` pairs. Decode the `pc`s:

   ```bash
   xtensa-esp32s3-elf-addr2line -pfiaC -e build/app.elf 0x4037b38b 0x4200af79
   # 0x4037b38b: _esp_error_check_failed at .../esp_system/esp_err.c:38
   # 0x4200af79: app_main at .../main/main.c:129 (discriminator 1)
   ```

2. **Make it reproduce.** Script the trigger as an automation scenario:

   ```yaml
   steps:
     - wait-serial: 'app: ready'
     - set-control: { part-id: btn1, name: pressed, value: 1 }
     - delay: 50ms
     - set-control: { part-id: btn1, name: pressed, value: 0 }
   ```

   `wokwi-cli . --scenario repro.yaml --fail-text 'Guru Meditation Error'` exits 1 when the
   text appears, 0 when the scenario completes, 42 at `--timeout` (default 30 s simulated).
   Keep the scenario in CI as the regression test once the bug is fixed, with the serial log
   (`--serial-log-file serial.log`, Action input `serial_log_file`) uploaded `if: always()`.

3. **Trace the bus.** Add a `wokwi-logic-analyzer` to `diagram.json`, wire `D0`..`D7` to the
   signals (SDA/SCL, MOSI/MISO/SCK/CS, TX/RX) and `GND` to ground, run with `--vcd-file
   capture.vcd`, open in PulseView (decoders for UART, I2C, WS2812), GTKWave or Surfer.
   `triggerMode: edge` with `triggerPin`/`triggerLevel` starts recording at the moment of
   interest.

4. **Attach GDB.** Start paused, connect with the GDB that matches the chip:

   ```bash
   wokwi-cli . --gdb-server-port 3333 --timeout 0
   # GDB server listening on port 3333. Simulation is paused until gdb connects: ...
   xtensa-esp32s3-elf-gdb build/app.elf -ex 'target remote localhost:3333'
   (gdb) break esp_system_abort     # every abort/assert/ESP_ERROR_CHECK failure stops here
   (gdb) continue                   # Thread 1 hit Breakpoint 1, esp_system_abort (...)
   (gdb) bt    # #2 _esp_error_check_failed (rc=-1, file="./main/main.c", line=129, ...)
   ```

   `break app_main` steps from the top; `--timeout 0` disables the 30 s timeout that would
   end the session. VS Code drives the same server: `gdbServerPort = 3333` in `wokwi.toml`, a
   `cppdbg` launch config with `miDebuggerServerAddress: localhost:3333` and `miDebuggerPath:
   ${command:espIdf.getToolchainGdb}`, "Wokwi: Start Simulator and Wait for Debugger", F5.

## Pitfalls

- **`--fail-text 'Guru Meditation Error'` misses aborts**: an `abort()`, `assert` or
  `ESP_ERROR_CHECK` panic prints no such line, so the run loops and exits 42 at the timeout.
  One string only; `--fail-text 'Rebooting...'` catches every panic (default panic config).
- **Start Wokwi before GDB**; a debugger started first finds nothing to connect to.
- **`Remote 'g' packet reply is too long`**: wrong GDB; C3/C6 need `riscv32-esp-elf-gdb`.
- **`gdbServerPort` in `wokwi.toml` pauses every run**, CI included, until GDB connects. The
  CLI auto-creates such a `wokwi.toml` for ESP-IDF projects; drop the line, use the flag.
- **Hang, not crash?** `Task watchdog got triggered` in the log lists the tasks that starved.

Docs: https://docs.wokwi.com/wokwi-ci/cli-usage#debugging-with-gdb. Demo with a VS Code
debug config: https://github.com/wokwi/esp32-idf-hello-wifi.
