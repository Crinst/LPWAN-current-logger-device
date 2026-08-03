# ADR-003: Network connectivity

**Status: OPEN — decision pending**
**Implements in:** firmware repo branch per ADR-001; board rev in this repo if ESP32 added

## Context

- Goal: "working Ethernet" + "possibly ESP32 for network connectivity".
- Legacy state: V3 board has ETH RMII + LAN8742 PHY wired and `MX_ETH_Init()` called,
  but **no TCP/IP stack was ever integrated** (no lwIP in the tree) — Ethernet logging
  is a stub (defect S10).
- User has strong ESP-IDF experience (ESP32 EVSE/cable projects) — an ESP32 co-processor
  (WiFi + BLE + its own TCP/IP) is familiar territory; STM32-side lwIP+ETH would be new
  integration work on this codebase.

## Options

| Option | Pros | Cons |
|---|---|---|
| A. Native Ethernet (lwIP or FreeRTOS+TCP on STM32) | Hardware already on V3; single MCU; wired reliability for a lab instrument | lwIP+cache+DMA on H7 is notoriously fiddly; no WiFi/BLE |
| B. ESP32 co-processor (UART/SPI link, e.g. ESP-AT or custom ESP-IDF app) | WiFi+BLE+TCP offloaded; user's home turf; isolates network stack from measurement RT | Extra hardware + link protocol; adds a second firmware |
| C. Both: ETH native + ESP32 for wireless | Full coverage | Most work; likely overkill for first next-gen release |

## Recommendation (analyst)

**B first** (fastest path to remote logging given ESP-IDF experience, and it can bridge
to Ethernet via an ESP32 with ETH if wired is needed), **A later** once the RTOS
firmware is stable — the PHY is already on the board, nothing is lost by deferring.

## Decision

_(pending)_
