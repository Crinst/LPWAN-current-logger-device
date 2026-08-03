# ADR-006: ESP32-S3 as HMI/connectivity processor (future direction)

**Status: DEFERRED 2026-08-03 — revisit after the RTOS baseline (ADR-001..005) works**

## Context

Idea raised during architecture planning: split the system into
**STM32 = pure instrument** (acquisition + FPGA link, ranging, calibration, power
source, SD, precise timing) and **ESP32-S3 = the face** (LCD + LVGL + touch,
WiFi/BLE, cloud/MQTT), linked by a UART/SPI status+data protocol.

Why it is attractive (analysis 2026-08-03):

- Removes all LVGL/framebuffer RAM pressure from the F767 — the 320×240 (or larger,
  with touch) UI runs on the S3's RAM/octal-PSRAM with its mature LCD peripheral.
- Plays to the user's daily ESP-IDF expertise; wireless comes native.
- V3 board stays unchanged — no display pins needed beyond the link UART.
- Cleaner real-time isolation than pushing UI+net onto the measurement MCU.

Caveats:

- **ESP32-S3 has no Ethernet MAC** — wired ETH stays on the STM32 (Zephyr
  `eth_stm32_hal`) or needs a W5500 on the ESP side, or is deferred.
- Three firmwares to maintain (STM + FPGA + ESP32).
- The STM↔ESP protocol becomes a critical external contract — design it versioned
  from day one.

Related rejected variant: adding external PSRAM/SDRAM to the STM32 — on the F767
this requires the FMC bus (~40 pins), impossible on the LQFP100/V3 pin budget;
it is effectively the "H745 + SDRAM + LTDC respin" (ADR-001 option C) in disguise.

## Decision

Deferred. Current focus (user, 2026-08-03): FPGA + STM32 + original ST7528 LCD on
a working RTOS first ("then we can improve"). When the baseline works, revisit this
ADR — the UI layer on the STM side should stay thin/display-agnostic so the HMI can
migrate to an S3 without touching the instrument core.
