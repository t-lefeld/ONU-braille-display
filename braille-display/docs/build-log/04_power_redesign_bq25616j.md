# 04 — Power Redesign: BQ25616J Single-Cell Architecture

**Date:** 2026-11
**Type:** Architecture change

## Overview

The power subsystem has been redesigned around a single Li-ion cell, USB-C input, and a managed charger with power-path control. This supersedes the 2S LiPo + TP5100 + MT3608 architecture described in [Build Log 01](01_power_architecture.md).

## Why the change

The v1/v2 power design was a bench arrangement. Three things pushed it toward a product design:

1. **No power path.** With the TP5100 arrangement, the boost converters hung directly off the battery. There was no system node, so the device could not run from USB with a dead or absent cell.
2. **No thermal management.** The design uses a bare Li-ion cell, not a protected pack. Nothing was monitoring cell temperature during charge.
3. **Mismatched converters.** The two rails used different boost chips (a TPS61032 and a TPS61085), which meant two layout patterns, two sets of passives, and two sets of failure modes to reason about.

A fourth, smaller reason: the schematic had drifted. The charging section had been edited enough times that redrawing it from the reference design was cheaper than repairing it.

## New architecture

```text
[USB-C] ──► [BQ25616J] ◄──► [1S Li-ion 2000mAh + NTC]
                 │
                 ▼
             [SYS rail]
                 │
        ┌────────┴────────┐
        ▼                 ▼
 [TPS61085 12V]    [TPS61085 5V]
   Coil rail         Logic rail
```

### Charger: BQ25616J

Selected for three properties:

- **Narrow-VDC power path management.** The SYS node is regulated independently of the battery, so the device operates immediately on USB even with a deeply discharged or absent cell.
- **Standalone operation.** No host control, no I²C. Every setting is a resistor. The charging subsystem is therefore independent of firmware state — which matters for the same reason the decoder enable pull-downs matter: safe behavior should not depend on code having run.
- **JEITA thermistor monitoring.** Required, because the pack is a bare cell.

### Pin configuration

| Pin | Net |
|-----|-----|
| 24 VBUS | USB-C VBUS, 1 µF to GND |
| 1 VAC | Tie to VBUS |
| 2 ACDRV | No connect (no external OVP FET) |
| 23 PMID | 10 µF + 1 nF to GND |
| 19, 20 SW | 1 µH to SYS |
| 21 BTST | 47 nF to SW |
| 22 REGN | 4.7 µF (10 V) to GND |
| 15, 16 SYS | 10 µF; feeds both boost inputs |
| 13, 14 BAT | Battery +, 10 µF to GND |
| 11 TS | NTC divider, see below |
| 3, 4 D+/D− | USB-C DP1+DP2 / DN1+DN2 |
| 8 ILIM | 316 Ω to GND (≈1.5 A) |
| 10 ICHG | 681 Ω to GND (≈1 A) |
| 12 VSET | No connect = 4.20 V |
| 9 CE | GND (charging enabled) |
| 6 OTG | GND — **must not float** |
| 5 STAT | 10 kΩ to REGN |
| 7 PG | 10 kΩ to REGN |
| 17, 18, pad | GND |

Charge current follows `I = 677 / R`, so 681 Ω gives ≈0.99 A into the 2000 mAh cell — 0.5C. Input limit follows `I = 478 / R`, so 316 Ω gives ≈1.5 A. Precharge and termination are each 5% of ICHG automatically.

### Thermistor network

```text
REGN ──[ RT1 5.36k ]── TS ──┬──[ RT2 31.6k ]── GND
                            └──[ NTC 103AT ]── GND
```

RT2 and the thermistor sit in parallel from TS to ground. The datasheet calls for 5.3 kΩ and 31.14 kΩ for the 0–60 °C JEITA window; 5.36 kΩ and 31.6 kΩ are the nearest E96 1% values. Use 1% parts here — tolerance moves the temperature trip points directly.

The 103AT is bonded to the **cell body**, not the PCB. A thermistor reading board temperature provides a false sense of protection. This moves the battery connector from 2-pin to 3-pin (B3B-XH-A: BAT+, NTC, GND).

### Boost converters

Both rails are TPS61085. Same topology, same layout pattern; only the feedback divider, compensation, and output cap rating differ.

| Component | 12 V coil rail | 5 V logic rail |
|-----------|----------------|----------------|
| L | 3.3 µH, Isat ≥ 2 A | 3.3 µH, Isat ≥ 2 A |
| D | PMEG2010AEH | PMEG2010AEH |
| R1 | 158 kΩ | 54.9 kΩ |
| R2 | 18.2 kΩ | 18.2 kΩ |
| RCOMP | 47 kΩ | 13 kΩ |
| CCOMP | 1.6 nF | 3.3 nF |
| COUT | 2 × 10 µF, 25 V | 2 × 10 µF, 16 V |

Output voltage is `VS = 1.238 × (1 + R1/R2)`, giving 11.99 V and 4.97 V. FREQ tied to IN selects 1.2 MHz. The 12 V compensation values are the datasheet's own 12 V / 3.3 V-in / 1.2 MHz row; the 5 V column uses the datasheet default and should be tuned on the bench.

## Open risk: coil rail margin

The TPS61085 produces approximately **0.5 A at 12 V** from a single-cell input near 3.5 V. The driver devices are rated **0.5 A per channel**. These numbers are identical.

If a solenoid draws near its driver limit, the coil rail runs at its ceiling with no margin, and the one-solenoid-at-a-time architecture is the only thing preventing a problem. The working estimate of ~83 mA per solenoid leaves ample headroom — but that figure predates the geometry scaling, the core material change, and the wire gauge change.

**The number is calculated, not measured.** One wound coil and a bench supply settles it, and that is the first measurement scheduled once the coil winder is operational.

## Bare cell: what is and is not protected

The charger provides overdischarge latch-off (BATFET opens below the depletion threshold, resets only on adapter plug-in), system overcurrent protection, battery overvoltage clamping at 4% above regulation, and the JEITA window.

It does **not** protect against a fault on the battery leads outside the board, or a cell-level internal fault. Acceptable for a bench prototype. Not acceptable for a device handed to a user — a cell with an integrated protection module is the production answer.

## Layout notes

1. PMID bulk cap directly against PMID and GND, shortest possible copper, with the 1 nF beside it.
2. L1 as close to SW as possible, narrow trace, single layer, no plane underneath.
3. **Thermal pad soldered down with vias into the ground plane.** Without it the charger thermally regulates at 110 °C and charge current silently drops.
4. Analog and power grounds separate, joined only at the thermal pad.
5. On both boosts, the SW → diode → output cap loop is the critical path. All three on the same layer, tight.

## Key Takeaways

- Power path management is the difference between a bench supply and a product — the device should run when plugged in regardless of battery state.
- A standalone resistor-configured charger keeps safety behavior independent of firmware, for the same reason the decoder enable pull-downs do.
- Using the same converter for both rails costs nothing and removes a whole category of "which chip was this again" errors.
- A bare cell needs the thermistor. Bond it to the cell, not the board.
- When a schematic has been edited enough times to become untrustworthy, redraw it from the reference design instead of repairing it.
