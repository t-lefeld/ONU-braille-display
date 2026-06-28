# Braille Display — 5-Cell Solenoid Driver

A custom 5-cell refreshable Braille display built around a dedicated PCB, solenoid power architecture, and embedded firmware. Designed from scratch as a personal hardware project, with an emphasis on reliable power sequencing and compact cell-select logic.

![Status](https://img.shields.io/badge/status-in%20progress-yellow)
![License](https://img.shields.io/badge/license-MIT-blue)
![PCB](https://img.shields.io/badge/PCB-Fusion%20360-orange)

---

## Overview

This display drives 40 solenoids (8 per Braille cell × 5 cells) to produce refreshable tactile Braille output. Each cell is independently addressable via a 74HC2238D 3-to-8 decoder, replacing the shift register approach used in v1. Power is sourced from a 2S LiPo with a dedicated 9V solenoid rail (MT3608 boost) and a separate 5V logic rail, keeping switching noise off the control circuitry.

The v1 design suffered voltage rail collapse traced to shift registers floating high during powerup — energizing all 40 solenoids simultaneously. v2 eliminates this by design: the 74HC2238D enable pin is pull-down guarded, guaranteeing all outputs are deasserted until the firmware explicitly asserts them.

---

## Features

- 5 refreshable Braille cells, 8 solenoids each (40 total)
- Custom PCB designed in Fusion 360
- 2S LiPo with TP5100 charger IC
- MT3608 boost converter — fixed 9V solenoid rail
- Separate 5V logic rail — isolated from switching noise
- 74HC2238D 3-to-8 decoders for cell addressing (no shift registers)
- Pull-down guarded enable pins — defined off-state at powerup
- BAT54S Schottky diodes for solenoid flyback protection

---

## Hardware

### System Block Diagram

```
[2S LiPo] ──► [TP5100 Charger] ──► [2S LiPo Pack]
                                          │
                         ┌────────────────┴────────────────┐
                         ▼                                  ▼
                  [MT3608 Boost]                     [5V Reg / Rail]
                  Fixed 9V Rail                      Logic Rail
                         │                                  │
              ┌──────────┤                         [MCU / Firmware]
              │          │                                  │
         [BAT54S ×40]    │                    [74HC2238D Decoders ×5]
         Flyback Diodes  │                                  │
              │          │                    ┌─────────────┘
              └──────────┼────────────────────┘
                         ▼
                  [Solenoids ×40]
                  (8 per cell × 5 cells)
```

### PCB

Designed in Fusion 360. Gerbers, schematic PDF, and BOM export located in `/hardware/pcb/`.

Key design decisions:
- Solenoid rail and logic rail routed on separate copper pours
- Enable pin pull-downs on all 74HC2238D decoders (R-pack to GND)
- BAT54S placed close to each solenoid header to minimize flyback loop area
- TP5100 charger exposed via USB-C input

### Bill of Materials

See [`/bom/bom.csv`](bom/bom.csv) for the full sourcing list.

| Reference | Part | Value / PN | Qty | Function |
|-----------|------|------------|-----|----------|
| U1 | TP5100 | TP5100 | 1 | 2S LiPo charger |
| U2 | MT3608 | MT3608 | 1 | Boost converter, 9V solenoid rail |
| U3–U7 | 74HC2238D | 74HC2238D | 5 | 3-to-8 decoder, cell select |
| D1–D40 | BAT54S | BAT54S | 40 | Schottky, solenoid flyback |
| BT1 | 2S LiPo | 500mAh | 1 | Main power source |
| SW1 | Power switch | SPDT | 1 | Power rail enable |
| SOL1–SOL40 | Solenoid | TBD | 40 | Braille pin actuation |
| R1–R5 | Resistor pack | 10kΩ | 5 | Enable pin pull-down |

---

## Firmware

**MCU:** TBD  
**Language:** C++  
**Build system:** PlatformIO (planned)

The firmware manages cell addressing via the 74HC2238D decoders. Each cell is selected by a 3-bit address on the decoder inputs; the enable pin is asserted only during a valid write cycle, ensuring the solenoid rail is never accidentally energized across multiple cells simultaneously.

### Building & Flashing

```bash
git clone https://github.com/YOUR_USERNAME/braille-display.git
cd braille-display/firmware

# Build (PlatformIO)
pio run

# Flash
pio run --target upload
```

### Firmware Structure

```
firmware/
└── src/
    ├── main.cpp          # Entry point, init sequence
    ├── cell.cpp/.h       # Per-cell addressing and solenoid drive logic
    ├── power.cpp/.h      # Rail enable sequencing
    └── braille_table.h   # ASCII → Braille dot mapping
```

---

## Build Log

Documented decisions, failures, and changes throughout development.

| # | Entry | Topic |
|---|-------|-------|
| 01 | [Power Architecture](docs/build-log/01_power_architecture.md) | 2S LiPo rail design, boost converter selection |
| 02 | [v1 Failure Analysis](docs/build-log/02_v1_failure_analysis.md) | Shoot-through root cause — shift register powerup state |
| 03 | [v2 Redesign](docs/build-log/03_v2_redesign.md) | Decoder swap, Schottky diode selection, enable pin strategy |

---

## Lessons Learned

- **Shift registers have undefined powerup state.** Without pull-downs or explicit init, outputs can float high — on a solenoid rail this means all loads energize simultaneously, collapsing the rail. Never assume a defined low state.
- **Decoders are safer than shift registers for solenoid addressing.** The 74HC2238D enable pin can be pull-down guarded, guaranteeing a deasserted state before firmware runs.
- **Schottky diode selection matters for solenoid flyback.** The BAV99 (v1) is a dual small-signal diode — adequate Vf but slow recovery. BAT54S provides lower Vf and faster switching, reducing flyback spike energy on the rail.
- **Separate power rails prevent noise coupling.** Running solenoids and logic from a common rail lets switching transients corrupt logic supply. A dedicated boost rail for solenoids keeps the logic rail clean.

---

## Repo Structure

```
braille-display/
├── README.md
├── LICENSE
├── hardware/
│   ├── pcb/              ← Gerbers, schematic PDF, Fusion 360 exports
│   └── mechanical/       ← 3D print files (if any)
├── firmware/
│   └── src/              ← C++ source
├── docs/
│   ├── build-log/        ← Dated markdown build log entries
│   └── images/           ← Photos, scope captures, diagrams
└── bom/
    └── bom.csv
```

---

## License

MIT — see [LICENSE](LICENSE)
