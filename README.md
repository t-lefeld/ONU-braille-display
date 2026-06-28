# Braille Display — 5-Cell Solenoid Driver

A custom 5-cell refreshable Braille display built around a dedicated PCB, solenoid power architecture, and embedded firmware. Designed from scratch as a personal hardware project, with an emphasis on reliable power sequencing and compact cell-select logic.

![Status](https://img.shields.io/badge/status-in%20progress-yellow)
![License](https://img.shields.io/badge/license-MIT-blue)
![PCB](https://img.shields.io/badge/PCB-Fusion%20360-orange)

---

## Overview

This display drives 40 solenoids (8 per Braille cell × 5 cells) to produce refreshable tactile Braille output. Each cell is independently addressable through dedicated decoder logic, replacing the shift register approach used in v1. Power is sourced from a 2S LiPo with a dedicated 9V solenoid rail (MT3608 boost converter) and a separate 5V logic rail, keeping switching noise isolated from the control circuitry.

This project was developed as a custom refreshable Braille display and draws inspiration from prior work within the maker and assistive technology communities. During the research phase, several existing refreshable Braille projects were studied to better understand the mechanical and electrical challenges involved in electromechanical Braille systems, including the Hackaday project *Electromechanical Refreshable Braille Module*. While those projects informed the overall understanding of the problem space, the electronics architecture, PCB design, power distribution strategy, firmware implementation, and system integration documented in this repository were developed independently for this project.

The v1 design suffered voltage rail collapse traced to shift registers floating high during power-up, energizing all 40 solenoids simultaneously. v2 eliminates this failure mode by design through guarded decoder enable lines that ensure all outputs remain disabled until firmware explicitly enables them.

---

## Features

* 5 refreshable Braille cells, 8 solenoids each (40 total)
* Custom PCB designed in Fusion 360
* 2S LiPo with TP5100 charger IC
* MT3608 boost converter for a dedicated 9V solenoid rail
* Separate 5V logic rail isolated from switching noise
* Decoder-based cell addressing architecture
* Pull-down guarded enable pins for defined startup behavior
* BAT54S Schottky flyback protection diodes
* Modular firmware structure for future expansion

---

## Hardware

### System Block Diagram

```text
[2S LiPo] ──► [TP5100 Charger] ──► [2S LiPo Pack]
                                          │
                         ┌────────────────┴────────────────┐
                         ▼                                  ▼
                  [MT3608 Boost]                     [5V Reg / Rail]
                  Fixed 9V Rail                      Logic Rail
                         │                                  │
              ┌──────────┤                         [MCU / Firmware]
              │          │                                  │
         [BAT54S ×40]    │                    [Decoder Logic ×5]
         Flyback Diodes  │                                  │
              │          │                    ┌─────────────┘
              └──────────┼────────────────────┘
                         ▼
                  [Solenoids ×40]
                  (8 per cell × 5 cells)
```

### PCB

Designed in Fusion 360. Gerbers, schematic PDF, and BOM exports are located in `/hardware/pcb/`.

Key design decisions:

* Solenoid and logic rails routed on separate copper pours
* Enable pin pull-downs to guarantee a defined off-state during startup
* BAT54S diodes placed close to solenoid connectors to minimize flyback loop area
* USB-C accessible TP5100 charging interface
* Compact routing optimized for handheld use

### Bill of Materials

See `/bom/bom.csv` for the complete sourcing list.

| Reference  | Part          | Value / PN | Qty | Function              |
| ---------- | ------------- | ---------- | --- | --------------------- |
| U1         | TP5100        | TP5100     | 1   | 2S LiPo charger       |
| U2         | MT3608        | MT3608     | 1   | 9V boost converter    |
| U3–U7      | Decoder IC    | TBD        | 5   | Cell addressing       |
| D1–D40     | BAT54S        | BAT54S     | 40  | Flyback protection    |
| BT1        | 2S LiPo       | 500mAh     | 1   | Main power source     |
| SW1        | Power Switch  | SPDT       | 1   | System power control  |
| SOL1–SOL40 | Solenoid      | TBD        | 40  | Braille pin actuation |
| R1–R5      | Resistor Pack | 10kΩ       | 5   | Enable pin pull-downs |

---

## Firmware

**MCU:** TBD
**Language:** C++
**Build System:** PlatformIO (planned)

The firmware manages cell addressing through decoder-based selection logic. Cells are selected using shared address lines while enable signals ensure that only the intended outputs are active during a write cycle. Startup sequencing initializes all control lines before enabling the solenoid rail, preventing unintended activation.

### Building & Flashing

```bash
git clone https://github.com/YOUR_USERNAME/braille-display.git
cd braille-display/firmware

# Build
pio run

# Flash
pio run --target upload
```

### Firmware Structure

```text
firmware/
└── src/
    ├── main.cpp
    ├── cell.cpp
    ├── cell.h
    ├── power.cpp
    ├── power.h
    └── braille_table.h
```

---

## Build Log

Documented decisions, failures, and design changes throughout development.

| #  | Entry               | Topic                                              |
| -- | ------------------- | -------------------------------------------------- |
| 01 | Power Architecture  | 2S LiPo rail design and boost converter selection  |
| 02 | v1 Failure Analysis | Shift register startup failure investigation       |
| 03 | v2 Redesign         | Decoder architecture and power sequencing redesign |

---

## Lessons Learned

* Undefined startup states can cause catastrophic power-up behavior in high-current systems.
* Solenoid systems benefit from hardware-level safeguards rather than relying solely on firmware.
* Flyback protection placement is just as important as diode selection.
* Separating power domains reduces switching noise and improves reliability.
* Designing for failure modes early often prevents difficult debugging later.

---

## Acknowledgments

This project benefited from studying prior refreshable Braille designs published by the maker and assistive technology communities.

In particular, the following project served as a useful reference during the research phase:

**Electromechanical Refreshable Braille Module**
https://hackaday.io/project/191181-electromechanical-refreshable-braille-module

This repository does not reproduce that design. The electronics, PCB layout, power architecture, firmware, and implementation details presented here were developed independently as part of this project.

---

## Repo Structure

```text
braille-display/
├── README.md
├── LICENSE
├── hardware/
│   ├── pcb/
│   └── mechanical/
├── firmware/
│   └── src/
├── docs/
│   ├── build-log/
│   └── images/
└── bom/
    └── bom.csv
```

---

## License

MIT — see LICENSE.

