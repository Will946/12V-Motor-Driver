# TB6612FNG Motor Driver Board

A single-IC dual-channel DC motor driver board built around the Toshiba **TB6612FNG**. It accepts a 12V motor supply and 3.3V logic supply, and drives two motors (or other DC loads) via direct GPIO control — no PWM enable line or standby control required.

## Overview

This board breaks out one TB6612FNG dual H-bridge driver IC with all required support circuitry: power decoupling, a power-good LED indicator, and header connectors for power, control signals, and motor outputs. It's designed to be one repeatable "cell" in a multi-board or multi-instance motor control system — multiple copies of this board can be powered from a shared 12V/3.3V/GND distribution and controlled independently from a microcontroller.

## What It's For

Driving small DC motors (e.g. brushed DC motors, peristaltic pump motors, gear motors) that:
- Run on 12V
- Draw up to ~1.2A continuous / 3.2A peak per channel
- Only need simple on/off or forward/reverse control (no onboard speed ramping required, though PWM control is possible if rewired)

Each board drives **two independent motors** (Channel A and Channel B).

## Key Design Decisions

- **STBY tied directly to VCC** — the IC is always active, no software standby control. Simplifies control to just the IN1/IN2 pins.
- **PWMA and PWMB tied directly to VCC** — both channels run at full enable at all times. Motor on/off and direction are controlled purely through AIN1/AIN2 and BIN1/BIN2 logic levels, not PWM gating.
- **No GPIO required for STBY/PWMA/PWMB** — frees up microcontroller pins; only 4 GPIOs are needed per board (2 per channel).
- **Multiple parallel pins for VM and GND** at the board connectors — spreads motor current across multiple contacts to reduce resistance and heat at the interconnect.
- **Single pin for VCC** — logic current draw is negligible (a few mA), so no need for parallel contacts.

## Bill of Materials (Key Components)

| Ref | Part | Function |
|-----|------|----------|
| U1 | TB6612FNG (SSOP-24) | Dual H-bridge motor driver IC |
| C6 | 0.1 µF ceramic | High-frequency decoupling, VM rail (placed close to IC) |
| C7 | 10 µF | Bulk decoupling, VM rail |
| C8 | 0.1 µF ceramic | High-frequency decoupling, VCC rail (placed close to IC) |
| C9 | 10 µF | Bulk decoupling, VCC rail |
| R2 | 2.4 kΩ | Current-limiting resistor for power indicator LED (fed from VM) |
| D2 | LED | Power-on indicator, lit whenever VM is present |
| J1 | 5-pin socket | Motor output connector (AO1, AO2, BO1, BO2 + GND) |
| J2 | 5-pin socket | Control input connector (AIN1, AIN2, BIN1, BIN2 + GND) |
| J3 | 5-pin socket | VM power input connector (VM ×3, GND ×1 — parallel pins for current capacity) |
| J6 | 3-pin socket | VCC logic power connector (VCC ×2, GND ×1) |

## Pinout / Connector Reference

### J1 — Motor Outputs
| Pin | Net |
|-----|-----|
| 1 | BO1 |
| 2 | BO2 |
| 3 | GND |
| 4 | AO2 |
| 5 | AO1 |

### J2 — Control Inputs (from microcontroller)
| Pin | Net |
|-----|-----|
| 1 | AIN2 |
| 2 | AIN1 |
| 3 | GND |
| 4 | BIN1 |
| 5 | BIN2 |

### J3 — Motor Power Input (12V)
| Pin | Net |
|-----|-----|
| 1 | VM |
| 2 | VM |
| 3 | GND |
| 4 | VM |
| 5 | VM |

### J6 — Logic Power Input (3.3V)
| Pin | Net |
|-----|-----|
| 1 | VCC |
| 2 | VCC |
| 3 | GND |

## Operation

### Power-up
1. Connect 12V to J3 (VM) and 3.3V to J6 (VCC). Both share GND with the rest of the system.
2. The power indicator LED (D2) lights immediately whenever VM is present, confirming motor supply is live.
3. Because STBY, PWMA, and PWMB are hardwired to VCC, the IC is fully active as soon as VCC is applied — no initialization sequence or enable command is needed.

### Motor Control Logic

Each channel (A and B) is controlled by a pair of logic inputs:

| IN1 | IN2 | Output State |
|-----|-----|--------------|
| H | L | Motor runs forward |
| L | H | Motor runs reverse |
| L | L | Motor stopped (outputs low, coasting) |
| H | H | Short brake |

For simple on/off (single-direction) loads, tie the unused direction input low and drive only one input HIGH/LOW from the microcontroller to start/stop the motor.

### Control Pin Summary
- **AIN1 / AIN2** → Channel A direction/on-off control (from MCU GPIO)
- **BIN1 / BIN2** → Channel B direction/on-off control (from MCU GPIO)
- **STBY, PWMA, PWMB** → hardwired to VCC on this board, not exposed for external control

## Power Requirements

- **VM (Motor Supply):** 4.5V–13.5V, sized for 12V operation. Up to ~1.2A continuous / 3.2A peak per channel (2.4A/6.4A total for both channels, observing thermal limits).
- **VCC (Logic Supply):** 2.7V–5.5V, 3.3V intended. Draws only a few mA — no special supply requirements.
- **GND:** Common return for both VM and VCC. Shared across all connectors.

## Design Notes

- **Decoupling placement:** The 0.1 µF ceramic caps on both VM and VCC are placed as close as possible to the IC's power pins to suppress high-frequency switching noise. The 10 µF bulk caps support transient current demand during motor start and provide additional local energy storage.
- **Multiple VM/PGND IC pins** (VM1/VM2/VM3 and PGND1/PGND2) are all tied together on the PCB — this is standard practice for this package, distributing current across more pads to reduce localized resistance and heating.
- **Connector current sharing:** J3 (VM) and the GND return use multiple parallel pins specifically to handle motor current without overloading a single contact. J2 (control signals) and J6 (VCC) use single pins per net since signal/logic current is negligible.
- **No exposed thermal pad:** The SSOP-24 package has no bottom thermal pad, so copper traces may be routed underneath the IC footprint without conflict.

## Intended Use in a Larger System

This board is designed as a repeatable unit — multiple instances can be deployed in parallel, each independently controlled via its own J2 connector, while sharing a common 12V and 3.3V distribution through J3/J6. This allows a single microcontroller to drive many motor channels (4 GPIOs per board) without needing per-board enable/standby management.

## Manufacturing Notes

- Designed for fabrication at JLCPCB (or equivalent), standard 2-layer (or more) FR4 PCB.
- SSOP-24, 0.65mm lead pitch package — verify footprint pad pitch against datasheet dimensions regardless of footprint source (DigiKey vs. LCSC libraries may differ cosmetically but should match electrically if pitch and body width are correct).
- All SMD components are reflow-solderable (paste + hot plate/oven compatible).
