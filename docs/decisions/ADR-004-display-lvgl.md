# ADR-004: Display and graphics stack

**Status: OPEN — decision pending**
**Implements in:** firmware repo branch per ADR-001; board rev in this repo

## Context

- Goal: LVGL with a full graphic LCD, ~**320×240**.
- Legacy: ST7528 160×100 mono LCD over I2C1 (u8g2), plus 7 EXTI buttons; enclosure
  (Fusion 360 files in `enclosure/`) is dimensioned for that LCD + button set —
  a bigger display implies enclosure rework too.
- 320×240 @ 16 bpp = 150 KB per framebuffer; double-buffered = 300 KB. F767 (512 KB
  RAM) can hold one full buffer or LVGL partial buffers; H743/H745 (~1 MB) fits double
  buffering comfortably. None of the LQFP100 parts expose enough FMC pins alongside
  the existing peripheral set for a parallel RGB panel without a board respin — SPI
  displays avoid that.

## Options

| Option | Pros | Cons |
|---|---|---|
| A. SPI TFT (ILI9341/ST7789, 320×240) + LVGL partial buffers + DMA | Works on any MCU option incl. existing V3 (SPI4 is free after dropping legacy SD-SPI); minimal HW change | ~20–30 fps ceiling; CPU share for flushes |
| B. Parallel/FMC 8080 TFT + LVGL | Faster flushes | Needs many FMC pins → board respin, conflicts with ETH/SDMMC pin budget on LQFP100 |
| C. LTDC RGB panel (only if H7 ≥ LQFP144 respin per ADR-001-C) | Flicker-free, LVGL direct mode, nicest result | Ties display choice to the biggest hardware option; needs SDRAM or generous internal RAM budgeting |

## Recommendation (analyst)

**A** for bring-up regardless of MCU choice (cheap, decouples SW from HW schedule);
revisit **C** only if ADR-001 lands on the H745 respin.

## Decision

_(pending)_
