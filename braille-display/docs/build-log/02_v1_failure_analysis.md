# 02 — v1 Failure Analysis: Voltage Rail Collapse

**Date:** 2026-06  
**Type:** Failure analysis

## Symptom

On powerup, the solenoid rail collapsed from ~8V to approximately 2V before any firmware had executed. No solenoids were commanded. The collapse was immediate and repeatable.

## Initial Hypotheses

1. Flyback diodes (BAV99) undersized or too slow — rail pulled down by flyback energy
2. MT3608 boost converter insufficient current headroom
3. Short circuit on solenoid rail PCB routing
4. Decoupling capacitance insufficient

## Investigation

### Scope capture on powerup

Probing the 9V rail at powerup showed the collapse coincided exactly with the MCU VCC rail coming up — not with any firmware-commanded solenoid event. This ruled out hypotheses 1 and 2 (flyback and boost headroom), since no solenoids were being driven.

### Shift register output state at powerup

The v1 design used shift registers to address solenoids. On powerup, with VCC rising but before the MCU had executed any initialization code, the shift register outputs were in an undefined state — measured at logic high on all outputs.

With all shift register outputs high, all 40 solenoid drive signals were asserted simultaneously. The 9V rail attempted to energize all 40 solenoids at once, drawing far more current than the MT3608 could supply. Rail collapsed.

## Root Cause

**Shift register outputs float to logic high during powerup before firmware initialization.**

This is a well-known behavior of unbuffered CMOS shift registers: outputs are not guaranteed low until explicitly clocked or reset. The v1 design had no hardware mechanism to hold solenoid drive signals low during the powerup window.

## What Changed in v2

### Shift registers replaced with 74HC2238D decoders

The 74HC2238D is a 3-to-8 active-low decoder with an enable pin (`/E`). When `/E` is high (deasserted), all outputs are forced high-impedance regardless of the address inputs.

A 10kΩ pull-down resistor on each `/E` pin ensures the enable line is held low at powerup — keeping all decoder outputs deasserted until the MCU explicitly drives `/E` high to begin a write cycle.

This provides a hardware-guaranteed safe state during the entire powerup window.

### BAV99 → BAT54S flyback diodes

Not the root cause of the v1 collapse, but replaced opportunistically during the redesign. The BAT54S offers lower forward voltage and faster reverse recovery than the BAV99, reducing flyback spike energy returned to the rail during solenoid deenergization.

## Key Takeaways

- Never assume shift register outputs are low at powerup — they are undefined until initialized
- Any solenoid drive circuit needs a hardware-enforced safe state that holds through the entire MCU boot sequence, not just after firmware runs
- Pull-down guarded enable pins on decoders are a clean solution: the safe state is the default, and the MCU must actively assert enable to drive outputs
- Scope powerup behavior early — the collapse happened before firmware ran, which immediately narrowed the root cause
