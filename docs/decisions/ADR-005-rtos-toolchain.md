# ADR-005: RTOS and toolchains

**Status: OPEN — decision pending**
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

_(pending — can be decided piecemeal; sub-decision 4 is uncontroversial and will be
applied to every ng branch unless vetoed)_
