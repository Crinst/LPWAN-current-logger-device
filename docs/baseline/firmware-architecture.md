# Legacy firmware architecture

As-is description of the two firmware generations, compiled 2026-08-03. Source repos:
`LPWAN_Current_Logger_FW_F7` (main, Sept 2021) and `LPWAN_Current_Logger_FW_H7`
(main, Dec 2020 incl. recovered UART-debug line).

## F7 — the working instrument (STM32F767VIT6, custom V3 board)

**Model: bare-metal cooperative super-loop, no RTOS.** (The `RTOS` branch on GitHub is
*not* an RTOS port — it is one commit behind main with only compiler-optimization
changes.)

Scheduling: free-running 32-bit µs counter (**TIM5**) polled in `while(1)`; four
soft-timed tasks:

| Task | Period | Work |
|---|---|---|
| Measuring | `settings.measuringInterval` (µs) | `adc_sample_ads8691()` → `ranges()` auto-range → `triggerMenu()` |
| Sampling/logging | `settings.samplingInterval` (µs) | format `measuringNumber value` → console (USART6) and/or SD |
| Console/UI | 250 ms | idle: `screenInterface()` + `consoleInterface()`; measuring: abort-key watch |
| Heartbeat | — | LED blink |

Support: **TIM4** = 1 µs busy-wait `microDelay()`; EXTI for 7 navigation buttons;
SPI TxRx-complete and UART DMA/IDLE interrupts feeding flags that the loop spin-waits
on. Detailed flow diagrams: `docs/flowcharts/Vyvojovy_diagram_sw_*.pdf` (measuring,
LCD menu, UART menu, SD card, USB flash, PC link — recovered thesis Visio exports).

Peripheral map (see `pinmaps.md` for pins):

- **SPI1 + DMA** → ADS8691 ADC (direct, no FPGA), CONV/RVS/RST/ALARM on GPIO
- **I2C1** → ST7528 LCD (NHD-C160100, 160×100) via **u8g2** (`Drivers/u8g2`, custom
  `st7528.c` driver present but bypassed)
- **I2C4** → AT24C256 EEPROM (settings) + digital potentiometers (programmable power
  source), `PS_EN` enable GPIO
- **SDMMC1 + FatFs** → SD logging, RTC-stamped filenames `20YYMMDD_HHMM.txt`
  (`File_Handling.c`); legacy SPI-mode SD driver on SPI4 unused
- **USART6 @ 3 Mbaud, DMA circular RX + IDLE** → PC console/data channel
  (`uart.c` + `ringbuff.c` ring-buffer driver); **UART7 @ 3 Mbaud** secondary/debug
- **USB OTG FS host (MSC)** + **ETH (RMII)** — configured, logging paths stubbed
- **TIM1_CH2N** buzzer, RTC calendar, TIM7/TIM14 auxiliary

Data flow:

```
ADS8691 ──SPI1/DMA──▶ adc_sample_ads8691() ─▶ measuredValue (double)
   ▲ CONV/RVS GPIO         │
   └─ range GPIOs ◀─ ranges()/change_range()      (see analog-frontend.md)
                           │
          previousValues[SAMPLES=10] ring ─▶ averaging()/trigger
                           │
        console (USART6 3 Mbaud)   SD (FatFs/SDMMC1)   [USB, ETH: stubs]
```

State persistence: `struct deviceSettings` in AT24C256 EEPROM (I2C4).

## H7 — the abandoned dual-core restart (STM32H745ZI Nucleo)

**Model: CubeMX dual-core skeleton + two real subsystems.** ~15–20 % application code.

Core split:

- **CM7 (master)**: clock config, releases CM4 via HSEM, then acts as relay/printer —
  drains the CM4→CM7 ring buffer, echoes to CM7→CM4, transmits on USART3 (VCP debug)
  and USART1 (3 Mbaud, DMA — **the unresolved problem**; see `legacy/uart-dma-debug`
  branch). Owns ETH, USB (unused), SPI1 (configured, unused).
- **CM4 (slave)**: wakes from STOP on HSEM, owns measurement — **SPI2 + DMA** to
  ADS8691 (`external_adc.c`, ported subset of the F7 driver), **TIM16** PWM interrupt
  triggers each `HAL_SPI_TransmitReceive_DMA` sample, USART2 @ 3 Mbaud debug out.

Inter-core transport (the one solid piece — Tilen Majerle's design, reusable):

- Two lock-free SPSC ring buffers (`ringbuff.c`) in **SRAM4/D3 @ 0x38000000**
  (`Common/Inc/common.h`: `BUFF_CM4_TO_CM7_ADDR` / `BUFF_CM7_TO_CM4_ADDR`, 0x400 B
  data each, `MEM_ALIGN` layout); both cores map `volatile ringbuff_t*` to the same
  physical addresses.
- **HSEM used only for the boot handshake** (`RCC_FLAG_D2CKRDY` wait → FastTake/
  Release semaphore 0 → CM4 leaves STOP). Runtime data exchange is 250 ms polling;
  the `HSEM_CM4_TO_CM7`/`HSEM_CM7_TO_CM4` defines exist but are unused.

What was never ported: auto-ranging, offset calibration, display, SD, console menus,
settings/EEPROM. Ghosts of the F7 feature set survive as commented constants in
`external_adc.h` (`RANGE_UPPER/LOWER_LIMIT_NA/UA/MA`, `CHANGE_RATIO`,
`previousValuesRange[]`).

## Shared components worth carrying into next-gen

| Component | Where | Verdict |
|---|---|---|
| Ring-buffer UART DMA driver (`uart.c` + `ringbuff.c`) | F7 + H7 (Majerle lwrb-style) | Replace with vendor-maintained `lwrb` + HAL, same concept |
| Inter-core shared-RAM layout (`common.h`) | H7 | Sound design; add HSEM signalling + cache-safe (MPU non-cacheable region) |
| ADS8691 driver | F7 `main.c` + H7 `external_adc.c` | Extract into a proper module; two half-drivers exist today |
| FatFs + `File_Handling.c` helpers | F7 | Reusable as-is on SDMMC |
| u8g2 + ST7528 | F7 | Superseded by LVGL + new display in next-gen (ADR-004) |
