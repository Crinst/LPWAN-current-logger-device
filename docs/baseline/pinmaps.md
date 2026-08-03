# Consolidated pin maps (legacy baseline)

Extracted 2026-08-03 from `pins.pcf`/`top.v` (FPGA, consolidated reformating state),
the CubeMX pinout report `docs/DP_STM32F767VIT_LQFP100.txt`, and the `.ioc` files of
the H745 and F401RE projects.

## Naming note (recurs everywhere)

`ADC_RST`, `ADC_CONV`, and the ADC busy/ready line recur across all designs with
spelling drift: F767 = `ADC_RVS` (PD5), H745 = `ADC_RSV` (PE6), FPGA = `rvs` (PIN_7).
Same ADS8691 RVS signal. Next-gen should standardize on **RVS** (datasheet name).

## 1. TinyFPGA BX (iCE40 LP8K CM81) — `TinyFPGA_BX_SPI_Master_FSM`

| Net | Header | Ball | Dir | Function |
|---|---|---|---|---|
| CLK | — | B2 | in | 16 MHz board clock → PLL |
| USBPU | — | A3 | out | driven 0 (USB disabled) |
| LED | — | B3 | out | heartbeat blink |
| PIN_1 | 1 | A2 | out | test output |
| PIN_2 | 2 | A1 | out | MCU-SPI SCLK |
| PIN_3 | 3 | B1 | out | MCU-SPI MOSI |
| PIN_4 | 4 | C2 | in* | MCU-SPI MISO |
| PIN_5 | 5 | C1 | out | MCU-SPI CS |
| PIN_6 | 6 | D2 | in* | MCU transfer-ready / data-send ACK (`w_adc_master_mcu_transfer_ready`, gates MCU-packet transmission) |
| PIN_7 | 7 | D1 | in* | ADC RVS (busy/ready) |
| PIN_8 | 8 | E2 | in* | ADC-SPI MISO |
| PIN_10 | 10 | G2 | out | ADC-SPI MOSI |
| PIN_11 | 11 | H1 | out | ADC-SPI SCLK |
| PIN_12 | 12 | J1 | out | ADC-SPI CS (gated by FSM `cs_control`) |
| PIN_13 | 13 | H2 | out | ADC CONVST — **driver commented out (undriven!)** |
| PIN_14 | 14 | H9 | out | range mA — power transistor |
| PIN_15 | 15 | D9 | out | range µA — power transistor |
| PIN_16 | 16 | D8 | out | range nA — power transistor |
| PIN_17 | 17 | C9 | out | range mA — analog switch (inverted) |
| PIN_18 | 18 | A9 | out | range µA — analog switch (inverted) |
| PIN_19 | 19 | B8 | out | range nA — analog switch (inverted) |

`*` **Port-direction inconsistency carried from legacy**: `top.v` contains `assign`
statements driving PIN_4, PIN_7 and PIN_8 although they are declared `input`
(and comments for PIN_6/PIN_7 disagree with usage). Must be fixed before next
synthesis — see defect F2 in `known-defects.md`. PIN_9 and PIN_20–31 defined in
the pcf, unused.

## 2. STM32F767VIT6 (LQFP100) — V3 board

### Measurement chain
| Pin | Port | Function | Label |
|---|---|---|---|
| 29 | PA5 | SPI1_SCK | ADC SPI |
| 88 | PD7 | SPI1_MOSI | |
| 90 | PB4 | SPI1_MISO | |
| 87 | PD6 | GPIO_Out | SPI1_CS |
| 86 | PD5 | GPIO_In | ADC_RVS |
| 91 | PB5 | GPIO_Out | ADC_CONV |
| 92 | PB6 | GPIO_Out | ADC_RST |
| 36 | PB2 | EXTI2 | Extra_GPIO (ADC ALARM) |

### Range switching
| Pin | Port | Label |
|---|---|---|
| 96 | PB9 | RANGE_MA |
| 97 | PE0 | RANGE_UA |
| 98 | PE1 | RANGE_NA |
| 1 | PE2 | ASW1 |
| 3 | PE4 | ASW2 |
| 4 | PE5 | ASW3 |
| 2 | PE3 | ASW4 |

### Buses & storage
| Pin(s) | Function | Used for |
|---|---|---|
| 95/93 (PB8/PB7) | I2C1 SCL/SDA | ST7528 LCD (+ LCD_RST = PA10, pin 69) |
| 59/60 (PD12/PD13) | I2C4 SCL/SDA | AT24C256 EEPROM + digipots (power source), PS_EN = PA4 (pin 28) |
| 80,83,65,66,78,79 | SDMMC1 CK/CMD/D0–D3 | SD card; CD = PD0 (81), WP = PD1 (82) |
| 42/43/44 (PE12/13/14) | SPI4 SCK/MISO/MOSI | legacy SPI-SD path, vestigial → **free for next-gen (e.g. SPI TFT)** |
| 63/64 (PC6/PC7) | USART6 TX/RX | PC console @ 3 Mbaud |
| 37/38/39 (PE7/PE8/PE9) | UART7 RX/TX/RTS | secondary @ 3 Mbaud |
| PA1,PA2,PC1,PA7,PC4,PC5,PB12,PB13,PB11 | ETH RMII | LAN8742 PHY (stack never integrated) |
| PA11,PA12,PA8,PA9,PB0 | USB OTG FS | host mode, VBUS switch + overcurrent EXTI |

### UI & misc
| Pin(s) | Function |
|---|---|
| PB1,PB14,PB15,PD8,PD9,PD10,PD11 | Buttons (EXTI): MEASURING, DOWN, PREV, NEXT, UP, ESC, ENTER |
| PE11/PE15/PB10 | LED BLUE/GREEN/RED |
| 40 (PE10) | TIM1_CH2N buzzer |
| PE6,PC13,PC2,PC3,PA0,PA3,PA15,PD3,PD4 | spare GPIO (PC2/PC3 → header JP12.8/JP12.7 — **analog-only `PC2_C/PC3_C` on H743, see ADR-001**) |

## 3. STM32H745ZI Nucleo (LQFP144) — dual-core test setup

| Port | Signal | Label | Core | Notes |
|---|---|---|---|---|
| PB10 | SPI2_SCK | | CM4 | ADC SPI, master, prescaler /16 |
| PC2_C | SPI2_MISO | | CM4 | |
| PC3_C | SPI2_MOSI | | CM4 | |
| PB11 | GPIO_Out | SPI2_CS | CM4 | software CS |
| PE15 | GPIO_Out | ADC_RST | CM4 | default high |
| PB8 | TIM16_CH1 PWM | ADC_CONV | CM4 | **CONVST driven by PWM** (presc 10, period 480, pulse 10) |
| PE6 | EXTI6 | ADC_RSV | CM4 | RVS as interrupt input |
| PB9 | GPIO_Out | TEST | CM4 | scope trigger |
| PD5/PD6 | USART2 TX/RX | | CM4 | 3 Mbaud debug |
| PB6/PB15 | USART1 TX/RX | | CM7 | 3 Mbaud data out (the unresolved DMA issue) |
| PD8/PD9 | USART3 TX/RX | STLINK VCP | CM7 | |
| PA5/PA6/PB5 | SPI1 | | CM7 | configured, unused |
| (RMII set incl. PG13/PG11) | ETH | | CM7 | Nucleo PHY, unused |
| PB0/PE1/PB14 | LD1/LD2/LD3 | | M7/M4/free | |

## 4. STM32F401RE Nucleo (LQFP64) — `F401RE_SPI_MasterSlaveTest`

| Port | Signal | Label | Notes |
|---|---|---|---|
| PC10 | SPI3_SCK | | full-duplex master, prescaler /32 |
| PC11 | SPI3_MISO | | |
| PC12 | SPI3_MOSI | | |
| PD2 | GPIO_Out | SPI3_CS | pull-up, init LOW |
| PA2/PA3 | USART2 TX/RX | VCP | debug/report channel |
| PA5 | GPIO_Out | LD2 | |
| PC13 | EXTI13 | B1 button | |

Wiring to the FPGA's MCU-SPI port (PIN_2..PIN_6) — see `protocol-fpga-mcu.md` for
the master/slave-role reconciliation.
