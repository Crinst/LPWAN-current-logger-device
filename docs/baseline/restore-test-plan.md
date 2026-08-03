# Hardware restore test plan (Phase 2)

Goal: prove the preserved 2021 FPGA↔MCU link still works on real hardware before any
next-gen work builds on it. Everything needed is in `~/GIT/LPWAN_Current_Logger_Legacy`.

## Hardware needed

- TinyFPGA BX
- Nucleo-F401RE
- **v2 prototype board** (carries the ADS8691 — this is what the 2021 FPGA tests
  used, dupont-wired), or skip ADC-side checks — the link framing test works
  without the ADC; packets then carry test-pattern bytes
- Jumper wires per the wiring table below; common GND

## Step 1 — Flash the preserved FPGA bitstream (do NOT rebuild first)

The archived `hardware.bin` is the exact binary that worked in Jan 2021. Its matching
source is the **archived Jan-17 top.v** (`bitstreams_20210117/from_MichalDell_PlatformIO/top.v`),
NOT any committed version (~318 diff lines vs commit 17659c3 — see RESTORE_NOTES.md).

```sh
pip install tinyprog          # TinyFPGA BX bootloader tool
cd ~/GIT/LPWAN_Current_Logger_Legacy/bitstreams_20210117/from_MichalDell_PlatformIO
tinyprog -p hardware.bin      # board in bootloader mode (press reset)
```

Expected: on-board LED blinks ~1 Hz (heartbeat) after programming.

## Step 2 — Flash the F401RE test firmware

Open `~/GIT/LPWAN_Current_Logger_Legacy/F401RE_SPI_MasterSlaveTest` in STM32CubeIDE
(project is complete, `Debug/` even contains the last built `.elf` — flashing that
directly via STM32CubeProgrammer is the lowest-risk path; rebuilding with a modern
CubeIDE may regenerate code).

## Step 3 — Test each side SEPARATELY (do not wire them together)

Code analysis (see `protocol-fpga-mcu.md`) established that **both sides are SPI
masters** — the F401RE program is a standalone MCU-side bring-up test, not the other
end of the FPGA link. Never connect the two SCLK push-pull drivers together.

**3a. F401RE standalone:** run it as-is; USART2 VCP (115200) must print
`Sending -- [0]: …` at init and, with PC11 (MISO) jumpered to PC12 (MOSI), each
`Receive` line must echo the TX pattern (63, 62, 61, …) — proves SPI3 + IT + UART
chain. Known quirks: `spiTxBuffer[0]` is uninitialized (off-by-one fill) and CS is
raised before the IT transfer completes — expect those artifacts, they are in the
legacy code, not your setup.

**3b. FPGA standalone (logic analyzer on PIN_2/3/5/6, ADC port PIN_10/11/12):**
after power-up the FPGA should autonomously clock 4-byte frames on the ADC port
(first frame = config `D0 14 00 0B`) and, once PIN_6 (MCU transfer-ready ACK input)
is tied high, burst 40-byte frames on the MCU port (~500 kHz SCLK on the Jan-2021
bitstream). Without an ADC, sample bytes are whatever MISO floats to — the framing
and CS cadence are what you're verifying.

**3c. (With the v2 board's ADS8691 dupont-wired to the ADC port + a current source):** forcing the
input across a range threshold must toggle range outputs PIN_14/15/16 (and inverted
PIN_17/18/19). ⚠ Note the consolidated build never routes CONVST to PIN_13 (assign
commented out) — check how the Jan-17 bitstream handles CONVST on the analyzer
before trusting conversions.

## Step 4 — Pass criteria

1. 3a passes: echo matches TX pattern.
2. 3b passes: correct frame sizes (4 B ADC / 40 B MCU), config word `D0 14 00 0B`
   visible on first ADC frame, range field in bits [31:29] of each sample word
   encoded `100/010/001` (mA/µA/nA) per the Jan-17 build.
3. 3c passes: range pins switch at the raw-count thresholds (≥220000 up, ≤2100 down).
4. Archive analyzer captures into `LPWAN_Current_Logger_Legacy/measurements/` for
   regression comparison against the next-gen implementation.

## Step 5 — Only after Step 4 passes: rebuild from source

Rebuild the archived Jan-17 `top.v` with a modern toolchain (apio ≥ latest, or
yosys+nextpnr-ice40) and verify the rebuilt bitstream behaves identically to the
preserved one. This validates the toolchain before the next-gen FPGA rewrite starts.
Expect warnings — the source has known port-direction inconsistencies (defect F2);
if nextpnr errors on multiple drivers for PIN_4/7/8, that confirms the defect and the
fix (correct the assigns) must be re-verified against Step 4 behavior.

## Optional Step 6 — V3 instrument smoke test

If the V3 board still powers up: flash the F7 firmware, but **first neutralize
defects S1/S2** (forced mA range + SD format at boot) or use a sacrificial SD card.
Verify: LCD UI, console at 3 Mbaud, a known current on each range.
