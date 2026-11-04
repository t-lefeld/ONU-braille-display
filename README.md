# Braille Display — 5-Cell Solenoid Driver

A custom 5-cell refreshable Braille display built around a dedicated PCB, solenoid power architecture, and embedded firmware. Designed from scratch as a personal hardware project, with an emphasis on reliable power sequencing and compact cell-select logic.

![Status](https://img.shields.io/badge/status-in%20progress-yellow)
![License](https://img.shields.io/badge/license-MIT-blue)
![PCB](https://img.shields.io/badge/PCB-Fusion%20360-orange)

---

## Overview

This display drives 40 solenoids (8 per Braille cell × 5 cells) to produce refreshable tactile Braille output. Each solenoid acts on a bistable cam magnet, so a dot holds its position once set and the coil needs no holding current. Dots are addressed through decoder logic that energizes one solenoid at a time.

Power comes from a single Li-ion cell charged over USB-C, with separate boosted rails for the solenoids and for logic so that switching transients stay out of the control circuitry.

This project was developed as a custom refreshable Braille display and draws inspiration from prior work within the maker and assistive technology communities. During the research phase, several existing refreshable Braille projects were studied to better understand the mechanical and electrical challenges involved in electromechanical Braille systems, including the Hackaday project *Electromechanical Refreshable Braille Module*. While those projects informed the overall understanding of the problem space, the electronics architecture, PCB design, power distribution strategy, firmware implementation, and system integration documented in this repository were developed independently for this project.

### Why electromagnetic actuation

The dominant actuator in commercial refreshable displays is the piezoelectric bimorph. It is fast and reliable, but one is needed per pin, so cost scales linearly with cell count, and it requires an operating voltage in the 100–300 V range. Published component cost for a piezoelectric cell is approximately **$35 per cell, or $4.38 per dot** ([Runyan & Carpi, 2011](https://doi.org/10.1586/erd.11.47)). Retail 40-cell displays currently list between roughly $2,700 and $3,760, which works out to $67–$94 per cell.

An electromagnetic cell trades refresh speed for cost and for a far lower drive voltage — 12 V here instead of 100–300 V. Published electromagnetic designs report $14.60/cell ([Chen et al., 2023](https://doi.org/10.1109/TOH.2023.3241952)) and $42.74/cell ([Bettelani et al., 2020](https://doi.org/10.1109/TOH.2020.2970929)). In the latter, $26 of that $42.74 was commercial solenoids — which is the reason this project winds its own coils rather than buying them, and the reason the coil winder below is on the critical path.

### Project goal

The target is an instructional device for people learning the Braille code, not a general-purpose screen reader. Early Braille instruction works with single characters, contractions, and short words rather than connected prose, so a five-cell device is a deliberate scope decision rather than a limitation. The tradeoff is refresh throughput, discussed under [Driver Architecture](#driver-architecture).

---

## Current Status

| Area | State |
|------|-------|
| Driver topology | Settled — decoder + transistor array |
| Power architecture | **Redesigned** — see [Build Log 04](braille-display/docs/build-log/04_power_redesign_bq25616j.md) |
| Charging schematic | In revision around the BQ25616J |
| PCB layout | Pending power section finalization |
| Coil winder | **Blocking** — chuck holder needs redesign for the larger core |
| Solenoid characterization | Blocked on the winder. All coil parameters are currently calculated, not measured |
| Firmware | Scope defined, not implemented |
| Companion app | Not started |

**Critical path:** coil winder chuck holder → wind a test coil → measure it → finalize the power section → fabricate.

The project is the subject of a Work-in-Progress paper submitted to the **ASEE North Central Section 2027 Annual Conference**.

---

## Features

* 5 refreshable Braille cells, 8 solenoids each (40 total)
* Bistable cam magnet actuation — no holding current
* Custom PCB designed in Fusion 360
* Single-cell Li-ion with USB-C charging and power-path management (BQ25616J)
* Separate 12 V solenoid and 5 V logic rails, both from TPS61085 boost converters
* JEITA thermistor monitoring on a bare cell
* Decoder-based addressing — one solenoid energized at a time
* Pull-down guarded enable pins for defined startup behavior
* BAT54S Schottky flyback protection diodes
* Perkins-style 8-key input, grade 1 / grade 2 toggle
* Flash-resident word table for a self-contained drill and practice mode

---

## Hardware

### System Block Diagram

```text
[USB-C] ──► [BQ25616J Charger] ◄──► [1S Li-ion 2000mAh + NTC]
             power path, JEITA
                    │
                    ▼
                 [SYS rail]
                    │
         ┌──────────┴──────────┐
         ▼                     ▼
  [TPS61085 Boost]      [TPS61085 Boost]
    12V Coil Rail         5V Logic Rail
         │                     │
    [BAT54S ×40]        [MCU / Firmware]
    Flyback Diodes             │
         │              [Decoder Logic]
         │                     │
         └──────────┬──────────┘
                    ▼
             [Solenoids ×40]
          (one energized at a time)
```

### Power Architecture

The power subsystem was redesigned in November 2026. The earlier 2S LiPo + TP5100 + MT3608 arrangement is superseded; see [Build Log 04](braille-display/docs/build-log/04_power_redesign_bq25616j.md) for the full reasoning and the complete pin-level design.

Summary of the current design:

| Stage | Part | Notes |
|-------|------|-------|
| Charger | BQ25616J | Standalone, no host control. Narrow-VDC power path means the device runs on USB even with a dead or absent cell |
| Charge current | 681 Ω on ICHG | ≈1 A into a 2000 mAh cell (0.5C), per `I = 677 / R` |
| Input limit | 316 Ω on ILIM | ≈1.5 A |
| Battery temp | 103AT NTC + 5.36 kΩ / 31.6 kΩ | JEITA 0–60 °C window. Required because the pack is a bare cell |
| Coil rail | TPS61085 | 12 V, divider 158 kΩ / 18.2 kΩ |
| Logic rail | TPS61085 | 5 V, divider 54.9 kΩ / 18.2 kΩ |

Both rails use the same converter deliberately: one qualified layout pattern, one set of failure modes, fewer line items on the BOM.

> **Open risk.** The TPS61085 delivers roughly 0.5 A at 12 V from a single-cell input, and the driver devices are rated 0.5 A per channel. Those numbers are identical, so the coil rail has no margin if a solenoid draws near the driver limit. The working estimate of ~83 mA per solenoid leaves ample headroom, but that figure predates the current core material, geometry, and wire gauge. **It is calculated, not measured.** Resolving this is the first bench measurement once the winder runs.

### Driver Architecture

The driver went through three architectures before settling. The full reasoning is in [Build Log 05](braille-display/docs/build-log/05_driver_iteration_history.md).

| Iteration | Approach | Why it was rejected |
|-----------|----------|---------------------|
| 1 | Discrete H-bridge per dot | 40 bridges ≈ 160 switches and 80 logic signals. Does not fit the board or the pin budget, and creates 40 separate shoot-through opportunities |
| 2 | TI DRV8912-Q1 integrated 12-channel half-bridge | Fixes component count, but needs 7 devices for 40 dots, is automotive-qualified in a cost-reduction project, and its per-channel PWM and simultaneous drive go unused by a bistable cam |
| 3 | **Decoder + transistor array** (current) | — |

The decoder approach wins on a property the other two lack: a decoder asserts exactly one output *by construction*, so addressing two dots at once is not expressible. The remaining hazard sits at the enable level rather than the output level, and is handled in hardware — the enable lines are pull-down guarded so outputs stay deasserted through the entire powerup window, and a dead time of ~10 µs is enforced between enable transitions.

This is the same lesson the v1 failure taught, applied one level up: make the safe state the default in hardware instead of relying on firmware to establish it.

**The cost is throughput.** One dot moves at a time, so a full 40-dot refresh is strictly serial — on the order of two seconds at the ~50 ms per dot reported for comparable ferrite-core designs. For reference, the Orbit Reader 20 refreshes cell-by-cell in under 0.75 s. For a five-cell instructional device with a detent that holds state without power, this is acceptable. **It would not scale to a 20-cell reading display**, and should not be read as a general-purpose architecture.

### Coil Design

The solenoid is the one component that cannot be purchased, which is what makes the coil winder a hard dependency rather than a convenience.

| Parameter | Value | Status |
|-----------|-------|--------|
| Core | Fair-Rite 3067990861, 2.5 × 20 mm, 67 material | Selected |
| Wire | AWG 38 | Calculated — set by the 0.5 A per-channel driver limit |
| Cam magnet | N52 NdFeB, 2 × 1 mm | Selected |
| Coil resistance | — | **Not measured** |
| Current draw | ~83 mA est. | **Not measured** |
| Pull force | — | **Not measured** |

Geometry is scaled up from the 1 × 10 mm core of the reference design to relax fabrication tolerances and use more readily sourced parts. The scaling is non-uniform because it follows standard ferrite rod sizes rather than a designed ratio.

### PCB

Designed in Fusion 360. Gerbers, schematic PDF, and BOM exports are located in `/hardware/pcb/`.

Key design decisions:

* Solenoid and logic rails routed on separate copper pours, joined at a single star ground point
* Enable pin pull-downs to guarantee a defined off-state during startup
* BAT54S diodes placed close to solenoid connectors to minimize flyback loop area
* Charger thermal pad soldered down with vias — without it the BQ25616J thermally regulates at 110 °C and charge current silently drops
* Compact routing optimized for handheld use

### Bill of Materials

See `/bom/bom.csv` for the complete sourcing list.

| Reference  | Part          | Value / PN | Qty | Function              |
| ---------- | ------------- | ---------- | --- | --------------------- |
| U1         | BQ25616J      | BQ25616J   | 1   | 1S charger, power path |
| U2         | TPS61085      | TPS61085   | 1   | 12 V coil rail boost  |
| U3         | TPS61085      | TPS61085   | 1   | 5 V logic rail boost  |
| U4–U8      | Decoder IC    | 74HC238D   | 5   | Cell addressing       |
| D1–D40     | BAT54S        | BAT54S     | 40  | Flyback protection    |
| BT1        | Li-ion cell   | 2000mAh 1S | 1   | Main power source     |
| RT1        | Thermistor    | 103AT      | 1   | JEITA battery temp    |
| SW1        | Power Switch  | SPDT       | 1   | System power control  |
| SOL1–SOL40 | Solenoid      | Custom     | 40  | Braille pin actuation |

---

## Firmware

**MCU:** ESP32-C3 (migration to ESP32-S3 planned for native USB-OTG and onboard grade 2 translation)
**Language:** C++
**Build System:** PlatformIO (planned)

The firmware manages cell addressing through decoder-based selection logic. Cells are selected using shared address lines while enable signals ensure that only the intended outputs are active during a write cycle. Startup sequencing initializes all control lines before enabling the solenoid rail, preventing unintended activation.

### Learning Mode

The learning mode is what distinguishes this from a generic display:

* Perkins-style 8-key input, mirroring the mechanical Braille writer used in instruction so the input skill transfers directly
* Grade 1 / grade 2 toggle
* Flash-resident word table driving drill exercises
* Tactile confirmation of correct and incorrect responses through the solenoids themselves

BLE and full liblouis translation are deferred to v3 — neither is required for the instructional task, and both would substantially expand the firmware surface.

### Building & Flashing

```bash
git clone https://github.com/t-lefeld/ONU-braille-display.git
cd ONU-braille-display/braille-display/firmware

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
| 01 | [Power Architecture](braille-display/docs/build-log/01_power_architecture.md)  | 2S LiPo rail design and boost converter selection (superseded by 04) |
| 02 | [v1 Failure Analysis](braille-display/docs/build-log/02_v1_failure_analysis.md) | Shift register startup failure investigation       |
| 03 | [v2 Redesign](braille-display/docs/build-log/03_v2_redesign.md)         | Decoder architecture and power sequencing redesign |
| 04 | [Power Redesign](braille-display/docs/build-log/04_power_redesign_bq25616j.md)      | BQ25616J single-cell architecture, full pin-level design |
| 05 | [Driver Iteration History](braille-display/docs/build-log/05_driver_iteration_history.md) | H-bridge → DRV8912 → decoder, and why each was rejected |

---

## Lessons Learned

* Undefined startup states can cause catastrophic power-up behavior in high-current systems.
* Solenoid systems benefit from hardware-level safeguards rather than relying solely on firmware.
* The textbook answer to a component-level requirement can be the wrong answer at system scale — an H-bridge is correct for one reversible load and wrong for forty.
* A better-engineered part can still be the wrong part when its strengths address a problem the system does not have.
* Flyback protection placement is just as important as diode selection.
* Separating power domains reduces switching noise and improves reliability.
* Designing for failure modes early often prevents difficult debugging later.
* Manufacturing capability is a design constraint of equal standing with electrical requirements. This project has been gated for months by a mechanical fixture, and no amount of additional circuit work would have advanced it.
* Keep calculated and measured values visibly distinct in your own documentation. Tagging the coil parameters as unverified is what surfaced the coil rail margin risk — at the schematic stage rather than after fabrication.
* Check the numbers that *motivate* a design, not only the numbers inside it. Several widely repeated figures about Braille display cost did not survive contact with primary sources.

---

## Acknowledgments

This project benefited from studying prior refreshable Braille designs published by the maker and assistive technology communities.

In particular, the following project served as a useful reference during the research phase:

**Electromechanical Refreshable Braille Module**
https://hackaday.io/project/191181-electromechanical-refreshable-braille-module

This repository does not reproduce that design. The electronics, PCB layout, power architecture, firmware, and implementation details presented here were developed independently as part of this project.

### References

1. N. H. Runyan and F. Carpi, "Seeking the 'holy Braille' display: might electromechanically active polymers be the solution?," *Expert Rev. Med. Devices*, vol. 8, no. 5, pp. 529–532, 2011. [doi:10.1586/erd.11.47](https://doi.org/10.1586/erd.11.47)
2. M. Etezad, R. Joshi, and F. L. Cibrian, "Advancements in refreshable Braille display technology: A comprehensive survey," *Displays*, vol. 90, 2025. [doi:10.1016/j.displa.2025.103133](https://doi.org/10.1016/j.displa.2025.103133)
3. H. Chen et al., "A novel refreshable Braille display based on the layered electromagnetic driving mechanism of Braille dots," *IEEE Trans. Haptics*, vol. 16, no. 1, 2023. [doi:10.1109/TOH.2023.3241952](https://doi.org/10.1109/TOH.2023.3241952)
4. G. C. Bettelani et al., "Design and validation of the Readable device: A single-cell electromagnetic refreshable Braille display," *IEEE Trans. Haptics*, vol. 13, no. 1, 2020. [doi:10.1109/TOH.2020.2970929](https://doi.org/10.1109/TOH.2020.2970929)
5. Texas Instruments, "BQ25616/616J standalone 1-cell 3.0-A buck battery charger," datasheet SLUSDF7A, 2022.
6. Texas Instruments, "TPS61085 650-kHz, 1.2-MHz, 18.5-V step-up DC-DC converter," datasheet SLVS859B, 2014.

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
