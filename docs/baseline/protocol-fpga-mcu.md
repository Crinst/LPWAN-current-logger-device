# FPGA ↔ MCU / FPGA ↔ ADC interface specification (legacy)

Extracted 2026-08-03 from three sources that must not be conflated:

| Version | Where | State |
|---|---|---|
| **Bring-up build** (Jan 16 2021) | git commit `17659c3` | 40 B packets, debug constants on the wire |
| **Working build** (Jan 17 2021) | `LPWAN_Current_Logger_Legacy/bitstreams_20210117/` (`top.v` + `hardware.bin`) | 40 B packets, real data, command channel — **the state verified on hardware** |
| **Consolidated** (Aug 2021, reformating) | FPGA repo `master` after restore | 120 B packets, 60 MHz PLL — never hardware-verified |

## Clocking

- Board clock 16 MHz → `SB_PLL40_CORE` (DIVR=0, DIVF=59, DIVQ=4) → **60 MHz** fabric
  clock in the consolidated build (net misleadingly named `w_clk_240mhz`).
  The Jan-2021 builds ran directly on 16 MHz (no PLL).
- ⚠ Consolidated build fails timing at 60 MHz (~30 MHz closure — defect F1).

## SPI parameters

Both FPGA ports are **SPI masters** (`SPI_Master_With_Single_CS`, Nandland core),
mode 0 (CPOL=0, CPHA=0), 8-bit, MSB-first. SPI clk = f_fabric / (2 × CLKS_PER_HALF_BIT).

| Port | Jan-2021 builds (16 MHz fabric) | Consolidated (60 MHz fabric) |
|---|---|---|
| ADC (PIN_8/10/11/12) | CLKS_PER_HALF_BIT=16 → **500 kHz**, 4 B/CS | CLKS_PER_HALF_BIT=3 → **10 MHz**, 4 B/CS, CS_INACTIVE=4 |
| MCU (PIN_2/3/4/5) | CLKS_PER_HALF_BIT=16 → **500 kHz**, **40 B/CS** | CLKS_PER_HALF_BIT=3 → **10 MHz**, **120 B/CS**, CS_INACTIVE=25 |

(Several in-code frequency comments are stale; the numbers above are computed from
the parameters.)

## Sample word format (32 bit, MSB-first on the wire)

```
bits [31:29]  range code          bits [17:0]* ADS8691 18-bit sample
```

18-bit reassembly from the ADC's 3 data bytes: `(rx[23:16]>>6) | (rx[15:8]<<2) | (rx[7:0]<<10)`.

Range encoding **differs between builds** (breaking change never reconciled):
- Bring-up `17659c3`: one-hot bits `[31]`=mA, `[30]`=µA, `[29]`=nA
- **Working Jan-17 + consolidated**: encoded field `[31:29]` = `100` mA / `010` µA / `001` nA

## MCU packet

- Jan-2021: **40 bytes = 10 samples × 4 B**; consolidated: **120 bytes = 30 samples × 4 B**
  (`ADC_READINGS`), double-buffered ping-pong; sample *n* at offset `4n`, MSB first.
- Transfer starts when a buffer is full (`tx_data_ready`) **and** the MCU asserts the
  transfer-ready/ACK line (PIN_6 → `w_adc_master_mcu_transfer_ready`).
- `FPGA_ADDR = 0x55` / `ADC_ADDR = 0xAA` are declared in the consolidated build but
  **never used** — the addressed-packet protocol was an intention, not an implementation.

## MCU → FPGA → ADC command channel (working Jan-17 build only)

The verified-working build added a pass-through absent from both git snapshots'
functional paths: if the **5th byte** received from the MCU equals `0xAA`
(`ADC_WRITE_COMMAND`), the FPGA latches the first 4 received bytes and sends them
verbatim as the next 4-byte ADC SPI frame (e.g. host-initiated `D0 14 00 0B` config).
The consolidated build **dropped this** — MCU RX bytes are stored but unused.
**Next-gen must re-add a command channel; the Jan-17 build is the reference.**

## ADC sequencing (consolidated build, 60 MHz counts)

1. Conversion trigger every 1200 clocks (**~20 µs → ~50 kS/s**)
2. `ADC_SAMPLING`: CONVST high + CS held low for `ADC_CS_CONVS_DELAY = 10` (~167 ns),
   then CONVST low, wait `ADC_SAMPLE_DELAY = 45` (~750 ns; datasheet needs ≥666 ns)
3. `ADC_ACQUIRING`: 4-byte SPI exchange; first-ever frame sends config `0xD0 14 00 0B`
   (WRITE → RANGE_SEL 0x14 = 0x0B, 1.25×V_REF unidirectional), then zeros/commands
4. Every conversion feeds the range FSM; only every 6th
   (`ADC_MEASUREMENT_UNDERSAMPLE = 5`) is stored into the MCU packet →
   **~8.3 kS/s to the MCU, ~50 kS/s range-tracking**

⚠ `PIN_13` (CONVST output) has its `assign` **commented out** in the consolidated
build — CONVST never reaches the ADC as-is. The working Jan-2021 wiring must be
checked on the bench (defect: add to F-list).

## Range FSM (consolidated)

States CHECKED/UNCHECKED/DONE; codes mA=0, µA=1, nA=2 (default mA). Thresholds on
raw counts (compare `[23:0]`): up-switch at ≥ **220000** (nA→µA, µA→mA), down-switch
at ≤ **2100** (mA→µA, µA→nA), else hold. Outputs PIN_14/15/16 (transistors, active
high) + PIN_17/18/19 (analog switches, inverted). Decision on **every** conversion.

## F401RE test harness — what it actually is

`LPWAN_Current_Logger_Legacy/F401RE_SPI_MasterSlaveTest`: STM32F401RE, SPI3
**master** (42 MHz APB1 / prescaler 32 = **1.31 MHz**, mode 0, soft CS on PD2),
40-byte `HAL_SPI_TransmitReceive_IT` per 500 ms loop, results printed on USART2
@115200 (`Sending -- [0]…`, `Receive -- [0]…`).

**Reconciliation (confirmed by the author, 2026-08-03):** the F401RE program was an
**MCU-side bring-up/loopback test only**. The actual validated 2021 chain was:
**v2 prototype board (ADS8691) ⇠dupont⇢ TinyFPGA BX ⇠dupont⇢ V3 board spare
headers (JP12 area, F767)** — a wiring-level proof of concept; the F767-side
receiving code was never committed. The FPGA's verified counterpart (something
acting as SPI *slave*) therefore exists nowhere in git. Wire-level parameters agree (mode 0, 8-bit, MSB-first, 40 B frames) — only
the roles conflict. **Next-gen decision: pick one master (recommend MCU master +
FPGA slave port with data-ready IRQ line, the conventional arrangement).**

## Known quirks to fix in any rewrite

- Port-direction inconsistencies: `assign` statements drive declared-`input` pins
  PIN_4/7/8 (defect F2)
- Legacy 40 B builds: `AdC_RANGE_LIMIT` typo; range compare uses full 32-bit word
  (incl. range bits!) instead of `[23:0]` — consolidated build fixed both
- F401RE test: off-by-one buffer fill (`spiTxBuffer[64]` write, index 0 uninitialized);
  CS raised immediately after starting the IT transfer (not synchronized to completion)
