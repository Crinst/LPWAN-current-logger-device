# ADR-002: Role of the FPGA in the acquisition chain

**Status: DECIDED 2026-08-03 — Option A: keep the FPGA, full rewrite**
**Implements in:** `LPWAN_Current_Logger_FW_FPGA` branch `ng` (if kept)

## Context

- Legacy proved the concept: TinyFPGA BX (iCE40 LP8K) as SPI master to the ADS8691
  with autonomous mA/µA/nA range switching + hysteresis, batching samples into
  40 B (later 120 B) packets for the MCU (verified working Jan 2021 — bitstream and
  MCU-side test preserved in `LPWAN_Current_Logger_Legacy`).
- But: the FPGA link was **never integrated into the real firmware** (defect X1); the
  working F7 instrument drives the ADC directly.
- FPGA value proposition: deterministic sample timing at max ADC rate (1 MSPS),
  range-switch reaction latency independent of MCU load, deep buffering (BRAM:
  iCE40 LP8K has 32× 4 kbit EBR ≈ 16 KB), MCU decoupling.
- FPGA costs: second toolchain (yosys/nextpnr), the open timing-closure problem
  (~30 MHz vs 60 MHz PLL, defect F1), no testbench culture yet (F4), board space/power.
- Counterpoint: an H7 @ 480 MHz with a hardware-triggered SPI+DMA chain and MDMA can
  service 1 MSPS × 4 B without CPU involvement; range switching in an ISR is µs-scale.
  A dual-core H745 CM4 doing only acquisition gets close to FPGA determinism.

## Options

| Option | Pros | Cons |
|---|---|---|
| A. Keep FPGA (rewrite: FIFO/ping-pong BRAM buffers, testbenches, fix timing) | True determinism; offloads MCU fully; matches "FPGA with advanced buffers" goal; educational value | Most engineering effort; two firmwares to maintain |
| B. Drop FPGA — MCU timer-triggered SPI+DMA direct to ADC | Simplest system; F7 already proved direct drive; one codebase | Range-switch latency depends on IRQ latency; sampling jitter under RTOS load must be verified |
| C. Hybrid: CM4 core plays the "FPGA role" (H745) | Dual-core isolation without extra toolchain | Still shares memory bandwidth/power domain with CM7 |

## Recommendation (analyst)

Decide **after** ADR-001. If H745 (dual-core) is chosen, Option C covers most of the
FPGA's value at a fraction of the effort — keep the FPGA only if you *want* the FPGA
work (valid reason). If F7/H743 single-core is chosen, the FPGA (A) earns its place.

## Decision

**Option A — keep the FPGA as the acquisition front-end, rewritten from scratch.**
With the single-core F767 (ADR-001), the FPGA owns deterministic sampling,
autonomous ranging and deep buffering. Requirements for the rewrite (branch `ng`
in `LPWAN_Current_Logger_FW_FPGA`):

- Testbench-first (iverilog/cocotb) — no untested RTL (defect F4)
- Clock chosen to actually close timing (30 MHz until proven otherwise — defect F1)
- Fix port directions (F2), route CONVST (F7), re-add the MCU command channel (F8)
- MCU-link roles resolved: FPGA becomes SPI **slave** on the MCU port + data-ready
  IRQ line; MCU (F767) is master — resolves defect T3
- BRAM FIFO buffering (iCE40 LP8K: ~16 KB EBR) with explicit overflow policy
