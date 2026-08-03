# Known defects and technical debt in the legacy baseline

Defect register compiled 2026-08-03 from full-codebase analysis of all five legacy
repos. Severity: **H** = breaks function or corrupts data, **M** = wrong/unreliable
behavior in some conditions, **L** = hygiene/maintainability.

## FPGA — `LPWAN_Current_Logger_FW_FPGA` (state after `restore/consolidate` merge)

| # | Sev | Where | Defect |
|---|-----|-------|--------|
| F1 | H | `TinyFPGA_BX_SPI_Master_FSM` | **Timing closure**: design is clocked from the 60 MHz PLL but place-and-route closes at only ~30 MHz (commit f2324ab message: "should be able to hit 30MHz effective speed based on timing simulation"). The chosen clock does not meet timing — behavior on silicon is at risk. Fix: pipeline the giant FSM `always` block or accept a 30 MHz clock. |
| F2 | H | historical `master` (pre-merge), `top.v` | MISO input pins were *driven* instead of read (`assign PIN_4 = w_spi_master_mcu_miso;` on an `input`, wire never fed from the pad); both SPI receive paths disconnected. Superseded by the reformating code, but **re-verify pin directions in the consolidated top.v before first synthesis**. |
| F3 | M | `top.v` | The `top.v` archived with the last working bitstream (`LPWAN_Current_Logger_Legacy/bitstreams_20210117/`, Jan 17 2021) contains ~318 diff lines never committed to git. The committed history does not exactly describe the hardware state that was verified working. |
| F4 | M | whole repo | **No testbenches, no simulation of any kind** — all validation was on hardware. Any rewrite must add iverilog/cocotb benches first. |
| F5 | L | `top.v` | Single ~1,500-line `always` block hosting 3 FSMs; misleading clock-frequency banner comments; `w_*`-named regs; blocking/non-blocking mix in clocked blocks; large commented-out regions. |
| F6 | L | repo | Build artifacts were committed historically (`.gitignore` added on `restore/consolidate`, 46ee666). |
| F7 | H | consolidated `top.v:930` | **CONVST never reaches a pin**: `assign PIN_13 = r_spi_master_adc_convst;` is commented out — the consolidated build cannot start ADC conversions as-is. |
| F8 | M | consolidated `top.v` | The MCU→FPGA→ADC command pass-through (0xAA channel) present in the verified Jan-17 build was dropped during the reformating rewrite — consolidated build ignores MCU RX bytes. |
| F9 | L | Jan-2021 builds | `AdC_RANGE_LIMIT` typo; range comparison uses the full 32-bit sample word (including the range bits) instead of `[23:0]` — fixed in consolidated build. |

## F401RE test harness — `LPWAN_Current_Logger_Legacy/F401RE_SPI_MasterSlaveTest`

| # | Sev | Defect |
|---|-----|--------|
| T1 | M | Buffer fill off-by-one: `for(i=64;i>0;i--) spiTxBuffer[i]=64-i;` writes out-of-bounds `spiTxBuffer[64]` and leaves index 0 uninitialized. |
| T2 | M | CS (PD2) is raised immediately after starting the 40-byte IT transfer, not on completion — CS timing unsynchronized. |
| T3 | H | Architectural: this test is SPI **master**, and so is the FPGA's MCU port — the two sides were never a matched pair (see `protocol-fpga-mcu.md`); a real link needs one side reworked as slave. |

## F7 firmware — `LPWAN_Current_Logger_FW_F7` (main, Sept 2021)

| # | Sev | Where | Defect |
|---|-----|-------|--------|
| S1 | H | `main.c:4696` area | `/***** TEST only *****/` block forces `currentRange = 2` (mA) at boot, overriding the configured nA default — shipped debug state. |
| S2 | H | `main.c:4890` area | Startup unconditionally **formats the SD card** (`Format_SD()`) and writes `FILE1.TXT`/`FILE2.TXT` test files — destroys previous logs on every boot. |
| S3 | M | `main.c:5140` | `endOfMeasurement == 0;` — comparison instead of assignment; the flag is never cleared. |
| S4 | M | `main.c:4858`, `1327`, `1364` | Bitwise `&`/`|` used as logical operators in conditions — works by accident on 0/1 flags, breaks on multi-bit values. |
| S5 | M | `main.c` | Duplicate function definitions differing only by signature (`convertInputToInt` at ~2213 & 2234; `getConsoleInput` at ~2322 & 2351) — mid-refactor state, ambiguous call sites. |
| S6 | M | UART/SPI paths | Blocking spin-waits (`while(isWaitingForData>0);`) around DMA/IT transfers — defeats the purpose of DMA, stalls the super-loop. |
| S7 | L | `main.c` (6,070 lines) | Monolith: ADC, ranging, EEPROM, power source, both UIs, storage and all CubeMX init in one file. Heavy dead/commented code (three display init variants, whole USB-mount block, SW-I2C paths). |
| S8 | L | `main.h` | Undocumented magic scaling constants (`0.8056640625`, `1.51`, `2.186`, `ADC_RESOLUTION 0.038146973`) — derivations only in the thesis (see `adc-ads8691.md`). |
| S9 | L | repo | `Debug/` build artifacts (`.o/.elf/.map`) committed; `fatfs_sd.c` (SPI-mode SD on SPI4) is vestigial — live path is SDMMC1. |
| S10 | L | features | USB-host logging and Ethernet logging are stubs: pins configured, `MX_ETH_Init()` called, but no lwIP stack and USB mount code fully commented out. `AUTORANGE_MODE 1` (linear-regression predictor) unfinished. One explicit TODO: HW alarm/trigger with hysteresis (`main.c:2139`). |

## H7 firmware — `LPWAN_Current_Logger_FW_H7` (main after `restore/merge-2020-lines`)

| # | Sev | Where | Defect |
|---|-----|-------|--------|
| H1 | H | CM7/CM4, shared buffers | **D-cache coherency**: shared ring buffers live in SRAM4 (0x38000000) and DMA buffers in cached regions; `SCB_CleanDCache_by_Addr` used inconsistently (present in CM7 `send_uart`, commented out on CM4). Latent data-corruption risk. |
| H2 | M | `CM7/Core/Src/main.c` loop | `HAL_UART_DeInit/Init(&huart1)` every loop iteration — debug leftover, thrashes the UART. |
| H3 | M | CM7 | `HAL_UART_TxCpltCallback` defined in both `main.c` and `uart.c` (one transmits "1234\n" on every completion — debug artifact). USART1 DMA TX was the unresolved problem when work stopped (see `legacy/uart-dma-debug` branch + patch in the Legacy archive). |
| H4 | M | `CM4/Core/Src/external_adc.c` | `adc_write_data()` / `adc_config()` declared `uint8_t` but never return a value — UB. |
| H5 | M | CM4 `main.c:318` | `ringbuff_write(..., "[CM4] Core ready\r\n", 19)` — 19 bytes for an 18-byte string, copies a stray byte. |
| H6 | L | both cores | Data path between cores is polled every 250 ms; the HSEM data-signalling defines in `common.h` exist but are unused (HSEM only used for boot handshake). |
| H7 | L | scope | Only ~15–20 % of the F7 application was ever ported: raw ADC read path + inter-core transport. Auto-ranging, calibration, display, SD, console — all absent. |

## Cross-cutting

| # | Sev | Defect |
|---|-----|--------|
| X1 | H | **The FPGA link was never integrated into the real firmware.** Only the standalone `F401RE_SPI_MasterSlaveTest` speaks the 40-byte protocol; F7/H7 firmware talk to the ADS8691 directly. The FPGA-in-the-middle architecture exists only as the FPGA side + a test harness. |
| X2 | M | Firmware and FPGA use different range-decision inputs/thresholds (FW compares scaled engineering values against 2.0/4.75/0.3 ratios; FPGA compares raw 18-bit counts against 2100/220000). Two independent implementations of the same feature — must be unified in next-gen (see `analog-frontend.md`). |
| X3 | L | Line-ending chaos (CRLF/LF mixed) across repos — normalize with `.gitattributes` in next-gen branches. |
