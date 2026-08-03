# ADC chain: ADS8910 vs ADS8691 — which is which

There are **two ADCs in this project's history**; conflating them causes real
confusion (they appear side by side in the firmware):

| | ADS8691 | ADS8910B |
|---|---|---|
| Role | First choice, **rejected** for the V3 board (integrated anti-alias filter, few-kHz cutoff, distorted fast signals — thesis p. 41). Lived on as the **FPGA-experiment ADC**: the **v2 prototype board** (which carries the ADS8691) was dupont-wired to the TinyFPGA (2020–21) | **Final ADC on the V3 board** (thesis §3.1.3) |
| Type | 18-bit SAR, 1 MSPS, single-ended, integrated AFE/ref | 18-bit SAR, 1 MSPS, fully differential, ±V_REF swing |
| Firmware path | `adc_sample_ads8691()` — active in F7 git head (Sept 2021) and H7 CM4 | `adc_sample()` ("NEW ADC") — active in the Jul-2020 thesis-era snapshot |
| Config used | register write `0xD0 14 00 0B` = WRITE → RANGE_SEL (0x14) = 0x0B (1.25×V_REF unidirectional) — used by FPGA + F7 `adc_config()` | thesis decoupling/ref network on V3 |
| Datasheets | on HDD: `04_HW - prototype doc/Datasheet/ads8691.pdf` | `ads8910b.pdf`, `ads8900b.pdf` (same family, 20-bit option) |

**Why git's "latest" F7 firmware uses the ADS8691 path:** the last commits (2021) were
made during the FPGA/ADS8691 experiment period — the firmware was re-pointed at the
experiment hardware. For reviving the **V3 instrument**, the ADS8910 path
(`adc_sample()`, as in the July-2020 thesis snapshot) is the correct one. The family
is pin-compatible 16/18/20-bit (ADS8920/8910/8900) — a 20-bit ADS8900 is a drop-in
resolution upgrade (thesis p. 41).

## Conversion sequence (both parts, as implemented)

1. Pulse **CONV** (F767: PB5 bit-bang; H745: TIM16_CH1 PWM; FPGA: FSM state)
2. Wait for **RVS** ready (F767 polls PD5; H745 EXTI on PE6; FPGA samples PIN_7)
3. Clock out 3–4 bytes of NOP/dummy over SPI (DMA on MCUs)
4. Reassemble 18-bit result:
   `value = (rx[2] >> 6) | (rx[1] << 2) | (rx[0] << 10)` (F7 `main.c:1902` area,
   identical logic in H7 `external_adc.c` and the FPGA bit-select)
5. Scale + offset-correct (below). ADS8910 bidirectional variant handles two's
   complement split at 0x1FFFF.

Register writes (`adc_write_data`): 4-byte frames `{cmd, regAddr, dataMSB, dataLSB}`
with manual CS.

## Scaling constants — derivations (thesis §3.4.2, Eq. 3.1)

**LSB = 2·V_REF / 2^N** with V_REF = 5000 mV:

| N | LSB | = firmware constant |
|---|---|---|
| 18 | 10 000 mV / 262 144 = **0.0381469726 mV** | `ADC_RESOLUTION 0.038146973` (mV/LSB) |
| 20 | 9.537 µV | (ADS8900 upgrade path) |

Because the front-end delivers exactly **1 mV per range-unit** (see
`analog-frontend.md`), LSB in µV maps 1:1 to milli-units of the active range:

| Range | Theoretical resolution (18-bit) | (20-bit) |
|---|---|---|
| nA | 38.15 pA (±19.07 pA quantization) | 9.54 pA |
| µA | 38.15 nA | 9.54 nA |
| mA | 38.15 µA | 9.54 µA |

Theoretical only — excludes tolerances, noise, offsets (thesis p. 56 disclaimer).
Other magic constants in `main.h` (`0.8056640625`, `1.51`, `2.186`) are empirical
per-range correction factors — **no derivation exists in the thesis**; treat as
calibration data to be re-derived properly in next-gen (per-unit calibration table
in EEPROM/flash instead of compile-time constants).

## Offset compensation

TMUX1111 channel 1 (grounded input) is selected and sampled to measure the chain's
DC offset (`settings.lastOffsetValue`), subtracted from readings. Next-gen: automate
as periodic auto-zero, store per range.

## Reference & supporting network (V3, thesis p. 41)

- **REF5050** 5.000 V (0.05 %, 3 ppm/°C) → RC low-pass (fc ≈ 482 Hz) → REFIN
- REFBUFOUT: series 1 Ω + 22 µF; supply 100 nF + 10 µF; 1 µF DECAP
- Input RC filters isolate the SAR sampling-cap kickback from the buffer op-amp

## Performance envelope

- ADC capable of **1 MSPS**; instrument spec requires logging down to **0.1 ms**
  period (10 kS/s). Legacy F7 firmware sustained ~kHz-class logging over the 3 MBaud
  console (see `LPWAN_Current_Logger_Legacy/measurements/`). The FPGA experiment
  targeted faster acquisition (10 MHz SPI @ 60 MHz fabric — timing-limited, defect F1).
