# Project documentation

This repo is the **head of the LPWAN Current Probe project**: hardware design files
plus the system-level documentation and decision log that steer the next-gen work.
Legacy firmware lives in its own repos (see table in `baseline/known-defects.md` /
`LPWAN_Current_Logger_Legacy/RESTORE_NOTES.md`); next-gen work streams are cut as
branches in the repo each decision points at.

## `baseline/` — the as-is system (extracted 2026-08-03)

| Doc | Contents |
|---|---|
| [`analog-frontend.md`](baseline/analog-frontend.md) | Shunts, gain chain, range switching, offset channel, power source, rails (from thesis + V3 schematic) |
| [`adc-chain.md`](baseline/adc-chain.md) | ADS8910 vs ADS8691 disambiguation, conversion sequence, LSB/scaling derivations |
| [`protocol-fpga-mcu.md`](baseline/protocol-fpga-mcu.md) | FPGA↔ADC and FPGA↔MCU interface spec across all three build generations; F401RE harness reconciliation |
| [`firmware-architecture.md`](baseline/firmware-architecture.md) | F7 super-loop and H7 dual-core architectures; reusable components |
| [`pinmaps.md`](baseline/pinmaps.md) | Complete pin tables: FPGA, F767/V3, H745 Nucleo, F401RE |
| [`known-defects.md`](baseline/known-defects.md) | Defect register across all legacy code (severity-rated, file:line) |
| [`restore-test-plan.md`](baseline/restore-test-plan.md) | Phase-2 bench procedure to re-verify the 2021 hardware state |

## `decisions/` — architecture decision records (all OPEN)

| ADR | Decision |
|---|---|
| [ADR-001](decisions/ADR-001-mcu-platform.md) | MCU platform (F767 / H743 drop-in / H745 respin / Nucleo dev) |
| [ADR-002](decisions/ADR-002-fpga-role.md) | Keep, drop, or replace the FPGA in the acquisition chain |
| [ADR-003](decisions/ADR-003-networking.md) | Native Ethernet vs ESP32 co-processor |
| [ADR-004](decisions/ADR-004-display-lvgl.md) | Display hardware + LVGL integration path |
| [ADR-005](decisions/ADR-005-rtos-toolchain.md) | RTOS choice and build toolchains |

Once an ADR is decided, record the decision in its file together with the repo +
branch that implements it.

## Other folders

- `eagle/` — schematics/boards v1–v3 + V3 extension, production gerbers, PCBWay CAM
- `enclosure/` — Fusion 360 / STEP / STL, slicer screenshots
- `ltspice/` — front-end amplifier simulations (MAX4239 saturation-recovery study)
- `bom/` — final BOM
- `board_pictures/`, `flowcharts/` — reference images and thesis-era SW flow diagrams
