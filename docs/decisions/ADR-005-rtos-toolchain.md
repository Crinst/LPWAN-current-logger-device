# ADR-005: RTOS and toolchains

**Status: DECIDED 2026-08-03 — Zephyr RTOS + west/CMake**
**Implements in:** all next-gen branches

## Context

- Goal: RTOS on the STM32 (legacy is a bare-metal super-loop; the "RTOS" branch name
  in the F7 repo never contained an RTOS).
- Legacy toolchains: STM32CubeIDE (Eclipse) for MCU; **apio** (Atom-era, effectively
  dead upstream in that form) for the iCE40.
- Natural task split for RTOS: acquisition (highest prio, or on CM4), storage/SD,
  UI/LVGL (LVGL wants a periodic handler + lock discipline), console/shell, network.

## Sub-decisions and options

1. **RTOS**: FreeRTOS via CMSIS-RTOS2 (CubeMX-generated, ST-supported, huge community)
   vs bare FreeRTOS vs Zephyr (bigger jump, own build system + drivers).
   *Analyst lean: CMSIS-RTOS2/FreeRTOS — smallest distance from the existing HAL code.*
2. **MCU build system**: stay CubeIDE vs **CubeMX-generated CMake + VS Code / CLI**.
   *Analyst lean: CMake — CI-able, Claude-Code-friendly, CubeMX still owns pin/clock
   config; matches how the user already builds ESP-IDF projects in VS Code.*
3. **FPGA toolchain** (if ADR-002 keeps the FPGA): modern **apio** release or plain
   **yosys + nextpnr-ice40 + icestorm** Makefile, plus **iverilog/cocotb testbenches
   as a hard requirement** (defect F4) and timing constraints checked in CI.
4. **Repo hygiene on ng branches**: `.gitattributes` (LF normalize, defect X3),
   build dirs ignored, no committed artifacts.

## Decision

**Zephyr RTOS with the west/CMake toolchain (VS Code editing, CI-able).**
Rationale: the project's feature set maps 1:1 to Zephyr subsystems — shell (replaces
the hand-rolled console menus), settings (replaces EEPROM handling), disk/FatFS
(SDMMC), one networking stack covering BOTH ADR-003 paths (native `eth_stm32_hal` +
`esp_at` ESP32 offload), integrated LVGL, logging. Future extension is cheap:
display swap = devicetree + driver change; H743 fallback = mostly SoC/DT change.

Accepted costs: custom board definition (devicetree) for the V3 board (reference:
upstream `nucleo_f767zi`, same MCU), and three custom drivers to write —
ADS8910/FPGA-link (SPI), ST7528 display (I2C), AD5259 digipot (I2C).

Sub-decisions: FPGA toolchain = yosys/nextpnr-ice40 + iverilog/cocotb testbenches
(per ADR-002); repo hygiene (.gitattributes LF, no artifacts) applies to all ng
branches. Zephyr app lives on branch `ng` of `LPWAN_Current_Logger_FW_F7` as a
west workspace application (board dir + app dir), keeping the branching model.
