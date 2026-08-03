# ADR-001: MCU platform for next-gen

**Status: OPEN — decision pending**
**Implements in:** firmware repo branch (see Decision), possibly new board rev in this repo

## Context

- Existing custom hardware: V3 board with **STM32F767VIT6** (LQFP100), working thesis
  firmware. Only custom board in existence; v1/v2 are Nucleo-driven front-ends.
- Verified (2026-08-03, V3 schematic vs STM32H753 datasheet + AN4936):
  **STM32H743VIT6 (LQFP100) is a near drop-in** for the V3 footprint — all used GPIO
  and supply pins align; only pins 17/18 differ (H7: `PC2_C/PC3_C` analog-only vs F7
  GPIO `PC2/PC3`), and on V3 those route only to spare header JP12 pins 7/8.
  Pre-swap checks: VCAP caps 2.2 µF, BOOT0 handling, USB supply per AN4936.
- **STM32H745 (dual core) has no LQFP100 package** (smallest LQFP144) → dual-core
  requires a board respin or the Nucleo-H745ZI devkit (where the 2020 bring-up +
  inter-core transport already run).
- Next-gen goals pulling toward more compute/RAM: RTOS, LVGL @ 320×240, Ethernet,
  SD, possibly FPGA link.

## Options

| Option | Pros | Cons |
|---|---|---|
| A. Keep F767 on V3 board | Zero hardware work; proven; 216 MHz + 512 KB RAM is enough for RTOS+LVGL on a modest display | Oldest platform; no dual-core isolation of acquisition |
| B. H743VIT6 drop-in on V3 | 480 MHz, 1 MB RAM, same board; cheap experiment | Rework/solder risk on the only board; single core |
| C. H745 dual-core, new board (LQFP144+) | CM4 = deterministic acquisition, CM7 = UI/net; existing inter-core code; best fit for goals | Full PCB respin (also the chance to fix front-end issues + add LCD/ESP32) |
| D. H745 Nucleo + V3/extension as front-end shield | No respin yet; dual-core benefits now; matches 2020 test setup | Frankenstein wiring; not a product form factor |

## Recommendation (analyst)

Start development on **D** (Nucleo-H745 + front-end) so firmware work is unblocked,
while designing **C** as the target board (which ADR-004/005 outcomes feed into).
Option B only if the V3 board becomes the long-term target.

## Decision

_(pending)_
