# 01 — Power Architecture

**Date:** 2026-06  
**Type:** Design decision

## Overview

Defined the full power architecture for the v2 Braille display, replacing the single-rail approach from v1 with a two-rail system: a dedicated 9V solenoid rail and a separate 5V logic rail.

## Requirements

- 40 solenoids, each rated ~9V
- Peak current draw when multiple solenoids fire simultaneously (worst case: full cell refresh)
- Logic supply must be noise-isolated from solenoid switching transients
- Rechargeable — no disposable batteries
- Compact form factor

## Design Decisions

### Battery: 2S LiPo

A 2S LiPo (nominal 7.4V, fully charged 8.4V) was chosen as the primary source. Advantages over a single-cell pack:
- Higher voltage headroom reduces boost converter duty cycle for the 9V rail
- Higher energy density for the physical size
- Widely available in compact form factors

Managed by a **TP5100** charger IC, which supports 2S charging directly with standard USB input.

### Solenoid Rail: MT3608 Boost Converter

The MT3608 boosts the 2S LiPo voltage to a fixed 9V solenoid rail. The output voltage is set via resistor divider on the feedback pin.

Key specs:
- Input: 2V–24V (comfortably covers 2S LiPo range)
- Output: up to 28V
- Peak current: 2A — adequate for staggered solenoid firing; simultaneous all-40 fire is avoided in firmware

The fixed 9V rail feeds solenoids through the decoder network only. It is not shared with logic.

### Logic Rail: 5V

A separate 5V rail powers all logic (MCU, decoders, pull-downs). Derived either from a linear regulator off the 2S pack or a second small buck — TBD based on quiescent current requirements.

Keeping the 5V rail separate from the MT3608 output means solenoid switching transients do not propagate to the MCU supply.

## Key Takeaways

- Two-rail separation (9V solenoid / 5V logic) is non-negotiable for noise immunity
- MT3608 is a simple, proven choice for fixed-voltage boost at this current level
- TP5100 handles 2S charging natively — no need for a separate cell-balance circuit
