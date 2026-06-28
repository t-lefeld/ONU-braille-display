# 03 — v2 Redesign

**Date:** 2026-06  
**Type:** Architecture change

## Overview

v2 is a ground-up redesign of the cell-select and power architecture based on the root cause identified in the v1 failure analysis. The core change is replacing shift registers with 74HC2238D decoders and hardening the power rail separation.

## Changes from v1

### Cell Addressing: Shift Registers → 74HC2238D Decoders

**v1:** Each solenoid was addressed via cascaded shift registers. On powerup, outputs were undefined (floated high), causing simultaneous energization of all solenoids.

**v2:** Each cell gets one 74HC2238D 3-to-8 decoder. The MCU sends a 3-bit address to select one of 8 solenoids within the active cell. A separate cell-select signal chooses which of the 5 cells is active.

The `/E` (enable) pin on each decoder is pulled low through a 10kΩ resistor to GND. The MCU must actively drive `/E` high to enable any outputs — meaning the solenoid rail is disconnected from all drive signals until firmware explicitly begins a write cycle.

Benefits over shift registers:
- Hardware-guaranteed off state at powerup (no undefined output)
- Simpler firmware — address + enable, no shift/latch cycle
- Fewer ICs for 5-cell addressing (5 decoders vs cascaded shift registers)

### Flyback Diodes: BAV99 → BAT54S

The BAV99 is a dual small-signal diode in SOT-23, commonly used for ESD and general-purpose clamping. It was selected in v1 for convenience.

For solenoid flyback the BAT54S is more appropriate:
- Lower forward voltage (Vf ~0.3V vs ~0.6V for BAV99 at low current) — less energy returned to rail per switching event
- Faster reverse recovery — reduces spike duration
- Same SOT-23 footprint — direct swap, no layout change needed

### Power Rail: Formalized Two-Rail Separation

v1 had notional separation between solenoid and logic rails but shared a common ground return path through the PCB in ways that allowed transient coupling. v2 PCB layout enforces:
- Separate copper pours for solenoid and logic returns, joined at a single star ground point
- Decoupling capacitors placed at each decoder VCC pin
- MT3608 output capacitor increased for better rail stiffness under pulse load

## v2 Component Summary

| Change | v1 | v2 | Reason |
|--------|----|----|--------|
| Cell select | Shift registers | 74HC2238D decoders | Safe powerup state |
| Enable control | None | Pull-down on /E pin | Hardware-enforced off |
| Flyback diodes | BAV99 | BAT54S | Lower Vf, faster recovery |
| Rail separation | Partial | Star ground, separate pours | Noise isolation |

## Key Takeaways

- The decoder + pull-down architecture makes safe powerup a hardware property, not a firmware responsibility
- BAT54S is the correct part for solenoid flyback — BAV99 is for signal-level clamping
- Star ground is worth the layout effort on mixed solenoid/logic boards
