---
name: find-memory-leaks
description: Find memory leaks in ESP-IDF firmware in CI. The Wokwi simulator watches the free heap while a scenario compresses hours of use into seconds and fails the run when the heap drops below a limit; no logging code in the firmware. Use for suspected leaks, soak, stress or endurance tests, or a device that runs out of memory after hours or days.
---

# Find memory leaks in ESP-IDF firmware

The simulator reads ESP-IDF's own heap counters out of the simulated chip, so the
firmware under test is the one you ship. Needs wokwi-cli 0.29 or newer, ESP-IDF 4.4 or
newer, and the ELF in `wokwi.toml` (`firmware = 'build/flasher_args.json'`,
`elf = 'build/my-app.elf'`).

1. **Generate the workload.** A request is whatever the product does repeatedly: a button
   press, a serial command, a sensor change. Scenarios have no loops, so script it:

   ```python
   import sys
   n = int(sys.argv[1]) if len(sys.argv) > 1 else 200
   print("name: Soak\nversion: 1\nsteps:\n  - wait-serial: 'app: ready'")
   for i in range(1, n + 1):
       print("  - set-control: { part-id: btn1, name: pressed, value: 1 }")
       if i == n:  # arm the proof right after the last press, before any delay
           print("  - wait-serial: 'handled %d requests'" % n)
       print("  - delay: 10ms\n  - set-control: { part-id: btn1, name: pressed, value: 0 }")
       print("  - delay: 40ms")
   ```

2. **Run once without a limit** and read the free heap after start-up settles:

   ```bash
   wokwi-cli . --scenario leak.test.yaml --timeout 60000 --heap-interval 500 --heap-csv heap.csv
   # [heap esp] 0.500s free=388008 ... Heap: 20 samples, free 388008 -> 388008, 0 bytes/s lost
   ```

3. **Pick the limit** below the settled value, leaving room for normal churn: a heap that
   settles at 380,000 with `--heap-min-free 350000` fails once about 30 KB is gone.

4. **Run with the limit** (add `--heap-min-free 350000` to that line). Exit 1 below it, 0
   when the scenario completes above it, 42 on timeout. `heap.csv` keeps every sample
   (`part,time_s,free,min_free`); the run fails on `free` only, `lowest` catches bursts.

   ```text
   [heap esp] 8.000s free=348944 min=347916
   Error: free heap 348944 dropped below 350000 bytes at 8.000s
   Heap: 16 samples, free 388008 -> 348944, lowest 347916, 5209 bytes/s lost
   ```

5. **Prove it bites.** The leaky build must fail and the fixed build pass; raise the
   request count until both hold. A soak test that never failed is not known to work.

6. **GitHub Actions:** use `wokwi/wokwi-ci-action@v1` inputs `heap_interval`, `heap_min_free`,
   `heap_csv` and `timeout: 60000` (default 10 s); upload `heap.csv` with `if: always()`.

## Pitfalls

- **No ELF, no samples.** ESP-IDF names it after `project(...)` in `CMakeLists.txt`.
- **A server without the feature passes the run** with only a warning. A green run
  without `[heap` lines is no result.
- **The limit is per build.** Chip, PSRAM, Wi-Fi and build options move it; derive it
  from step 2, never copy it.
- **Compress usage, not time.** `delay: 60m` hits the session cap; fire requests back to
  back. The simulated `--timeout` must cover boot plus every step (200 x 50 ms is 10 s)
  or the run exits 42.
- **A one-shot completion line is missed** unless the `wait-serial` comes right after the
  step that triggers it, with no delay between them (see step 1). No such line in the
  firmware? Log a count every 50 requests.
- **Restarts show as gaps, not drops.** Add `--fail-text 'Guru Meditation Error'`.
- **Non-ESP-IDF firmware takes no samples.** Log free memory from the firmware instead.

Docs: https://docs.wokwi.com/wokwi-ci/heap-monitoring. Demo (leaky handler, generator,
workflow, heap plot): https://github.com/wokwi/esp32-heap-leak-demo.
