# Analog measurement front-end (V3 board)

Source: thesis "Zařízení pro zaznamenávání proudové spotřeby LPWA senzorů"
(Mikulášek, VUT 2020), §3.1 + §3.4–3.5 (printed page numbers cited), cross-checked
against `eagle/v3/DP_current_measure_v3.sch`. Design target: **~5 nA … ~3000 mA** in
three ranges, sample period down to **0.1 ms**, max shunt drop **50 mV**, total gain
**A_U = 100** → 5 V ADC full scale (V_REF = 5.000 V).

## Range scaling — the core numbers

Every range delivers **1 mV at the ADC per unit of its range**:

| Range | Shunt | Part | Scaling | Full scale (50 mV drop) |
|---|---|---|---|---|
| nA | **10 kΩ** 0.05 % 10 ppm/°C (Panasonic ERA-6ARW103V) | 1 nA→10 µV ×100 = 1 mV/nA | 5 µA |
| µA | **10 Ω** 0.1 % 10 ppm/°C (TE RN73C2A10RBTD) | 1 µA→10 µV ×100 = 1 mV/µA | 5 mA |
| mA | **10 mΩ** 0.1 % Kelvin (Vishay Y14870R01000B9R, CSM2512) | 1 mA→10 µV ×100 = 1 mV/mA | 5 A (limited ~3 A) |

Each range usable ≈ 5…4900 units (p. 55). Inputs likely damaged above **~3 A**
(no dedicated terminal clamp — protection is design margin + 2 A PPTC on power input).

## Range switching (p. 37–38)

Shunts are paralleled onto the terminals by **N-MOSFETs** (relays rejected — too slow
for 0.1 ms period):

- **nA shunt permanently connected** (keeps terminals connected when device is off).
  Adds ≈0.1 % error to µA range, ≈0.0001 % to mA range.
- µA switch: **DMG3414UQ-7** (R_DS(on) ≈29 mΩ @3.3 V) → µA path ≈ 10.029 Ω.
- mA switch: **DMN2009USS** (R_DS(on) ≈9 mΩ) → mA path ≈ **19 mΩ vs 10 mΩ shunt** —
  the switch resistance is *not* negligible on the mA range; Kelvin-sense shunt
  mitigates but next-gen should calibrate this per unit.
- MCU/FPGA drive: 3 range-transistor lines + analog-switch lines
  (F767: `RANGE_MA/UA/NA` PB9/PE0/PE1 + `ASW1-4` PE2-PE5; FPGA: PIN_14-16 +
  inverted PIN_17-19 — see `pinmaps.md`).

**Note (thesis-era prototype V1 lessons, `docs/Prototyp_V1_notes.txt`):** the first
prototype used P-MOS with huge voltage drops and once destroyed a Nucleo through bad
range sequencing — the final N-MOS design fixed this, but range-switch sequencing
remains safety-relevant in firmware.

## Input multiplexer + offset channel (p. 38–39)

**TMUX1111** (4-ch SPST, ≤3 pA leakage, ~2 Ω, break-before-make) selects which shunt
voltage feeds the amplifier. Channels 2–4 = the three shunts; **channel 1 is tied to
ground and sampled to measure the chain's DC offset** for subtraction — this is the
hardware behind `settings.lastOffsetValue` in the firmware. Next-gen should keep and
automate this offset-cal path.

## Amplifier chain (p. 39–40)

Two non-inverting stages ×10 (**A_U = 100**) + unity buffer before the ADC S/H.

- First-stage candidate **MAX4239** (chopper, ≤2 µV offset) had **≤5.7 ms saturation
  recovery** → instability with switched inputs (LTspice sims in `ltspice/`).
- Chosen chopper for the chain: **ADA-class part "AS86288ARTZ"** (thesis p. 40;
  verify exact P/N against BOM — likely ADA4528 family): ≤1 µV offset, recovery
  ≤50 µs — fast enough for the 0.1 ms cycle.

## Auto-ranging (thesis §3.5.1 vs implementations)

Thesis specifies only the qualitative algorithm (up-switch above range's upper limit,
down-switch below lower limit, committed at end of range-check cycle; **no numeric
hysteresis given**). The two implementations diverged (defect X2):

| | F7 firmware (`ranges()`) | FPGA (range FSM) |
|---|---|---|
| Input | scaled engineering value (double) | raw 18-bit ADC counts |
| Thresholds | `RANGE_UPPER_LIMIT` 2.0, total 4.75, `CHANGE_RATIO` 0.3 (main.h) | `ADC_RANGE_LIMIT_LOW` 2100, `HIGH` 220000 |
| Equivalent | 2.0 units up / hold band ratio 0.3 | ≈0.08 mV / ≈8.39 mV at 38.15 µV/LSB |
| Mode | `AUTORANGE_MODE 0` last-value; mode 1 = unfinished linear-regression predictor (design data: `LPWAN_Current_Logger_Legacy/measurements/thesis_era/measure_range_change_alg_linear_regression.xlsx`) | hysteresis hold-band |

**Next-gen action:** define ONE range policy (units: raw counts, for speed) with
explicit hysteresis + dwell time, implemented once (wherever ADR-002 puts it).

## Programmable power source (p. 46–48)

- 2× **LT3045** in parallel (500 mA total, 0.8 µV_RMS noise, linear — no switching
  ripple into the measurement), 15 mΩ ballast each.
- 2× **AD5259BRM** digipots on I2C4 (nonvolatile, 256 steps):
  **0x18 = current limit** (10 kΩ + 330 Ω series → ≈14–455 mA, log-spaced),
  **0x4A = output voltage** (50 kΩ + 4.7 kΩ series → ≈0.47–5.47 V, ~0.02 V/step).
- Enable: `PS_EN` (PA4).

## Supply rails (p. 49–51)

| Rail | Regulator | Serves |
|---|---|---|
| 3.3 V digital | TLV76733 | MCU, digital |
| 5 V analog (low-noise) | LT1761ES5 (20 µV_RMS) | amplifier chain |
| 6 V | LK112M60TR | REF5050 headroom (needs ≥5.5 V in) |
| 5 V aux | TLV76750 | USB flash, digipot analog side |
| Reference | **REF5050** 5.000 V, 3 ppm/°C, 0.05 % | ADC V_REF (via RC filter, fc ≈ 482 Hz) |

## Comms isolation (p. 42–43, 56)

USB → **FT230XQ** UART bridge → **Si8642** digital isolator → MCU UART. Default
2 MBaud (settable 300 Bd–3 MBd). ESD: USBLC6-4SC6. (The 3 MBaud USART6 console in
firmware rides this isolated path.)
