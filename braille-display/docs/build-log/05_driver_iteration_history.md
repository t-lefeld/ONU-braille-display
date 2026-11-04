# 05 — Driver Iteration History

**Date:** 2026-11
**Type:** Design decision (retrospective)

## Overview

This entry records the full history of the solenoid driver architecture, which passed through three distinct approaches before settling. [Build Log 02](02_v1_failure_analysis.md) and [03](03_v2_redesign.md) cover the shift-register failure and the decoder swap; this entry is the wider arc, written retrospectively because the reasoning that eliminated each candidate turned out to be more useful than the final answer.

Each architecture was rejected for a **different** reason. That is the interesting part.

---

## Iteration 1 — Discrete H-bridge per dot

### The appeal

Polarity reversal is intrinsic to the bistable cam: raising a dot and lowering it are the same operation with opposite current direction. Considered as a single-actuator problem, an H-bridge is simply the correct answer.

### Why it failed

Considered at system scale it is not the correct answer at all.

| Problem | Magnitude |
|---------|-----------|
| Bridges required | 40 |
| Switching devices | ~160 |
| Logic signals if driven directly | 80 |
| Independent shoot-through opportunities | 40 |

The component count is incompatible with a board that fits a handheld enclosure. The pin count far exceeds any available microcontroller, which would have forced an expansion scheme — reintroducing exactly the complexity the discrete approach was meant to avoid.

And the hazard multiplies. Shoot-through occurs when the high-side and low-side devices of one leg conduct simultaneously. Forty independent bridges means forty separate opportunities for firmware to create that condition.

> **Rejected on:** component count and pin budget.

---

## Iteration 2 — TI DRV8912-Q1 integrated driver

### The appeal

The DRV8912-Q1 is a twelve-channel integrated half-bridge driver. On paper it solves everything iteration 1 got wrong:

- 12 half-bridges configuring into 6 full bridges
- Integrated overcurrent protection, open-load detection, per-channel diagnostics over SPI
- Entire control interface collapses to 4 wires, with daisy-chaining across devices
- 1 A per channel, 4.5–32 V supply

Measured against 160 discrete switches, this is a substantially better engineering answer, and it stayed on the table for a while.

### Why it failed

Three considerations, in increasing order of importance:

1. **Scale.** Six full bridges per device means **7 devices** for 40 dots. Each is a 24-pin HTSSOP at roughly 50 mm². That is a large fraction of the available board area.
2. **Cost.** An automotive-qualified driver array is hard to justify in a design whose entire premise is cost reduction relative to piezoelectric cells. Seven of them, more so.
3. **Capability mismatch — the decisive one.** The DRV8912 is built to drive many loads *simultaneously and independently*, with programmable PWM duty cycle per channel. A bistable cam needs a single short pulse of fixed polarity. No speed control, no position control, no simultaneity. The architecture eventually adopted energizes exactly one dot at any instant.

Paying in board area, cost, and supply-chain complexity for capability the mechanism does not use was not defensible.

> **Rejected on:** capability mismatch. The part was over-specified for the problem, not under-specified.

---

## Iteration 3 — Decoder + transistor array (current)

### The approach

A 3-to-8 CMOS decoder performs the addressing; transistor arrays do the switching. Enable pins are pull-down guarded so that outputs remain deasserted through the entire powerup window — the fix that came out of the [v1 failure analysis](02_v1_failure_analysis.md).

### Why it wins

The decisive property is structural: **a decoder asserts exactly one output by construction.**

Where iteration 1 offered forty separate chances for firmware to select conflicting states, the decoder matrix cannot address two dots at once *at all*. The fault is not prevented, guarded, or caught — it is inexpressible.

The hazard that remains sits one level up, at the enable inputs rather than the outputs, and it is handled the same way: in hardware. Enables are pull-down guarded, only one enable line is under software control, and a dead time of ~10 µs is enforced between transitions on it.

Flyback is handled by BAT54S dual Schottky diodes, chosen over BAV99 for lower forward voltage (see [Build Log 03](03_v2_redesign.md)).

### The cost

Throughput. One dot moves at a time, so a full 40-dot refresh is strictly serial — on the order of two seconds, at the ~50 ms per dot reported for comparable ferrite-core designs. The Orbit Reader 20, for comparison, refreshes cell-by-cell in under 0.75 s.

For a five-cell instructional device with a detent that holds state without power, and a use case that does not involve scanning connected prose, this is acceptable. **It would not be acceptable for a 20-cell reading display**, and this architecture should not be understood as scaling to one.

---

## A note on convergence

The reference design studied during the research phase ([Hackaday project 191181](https://hackaday.io/project/191181-electromechanical-refreshable-braille-module)) also uses a decoder-plus-transistor-array topology. This project arrived there independently, after exhausting the two alternatives above.

More interesting: discussion on that project proposed moving in the *opposite* direction, toward an integrated H-bridge driver of the DRV8212 family — precisely the path this project had already taken and abandoned.

Both positions are defensible, and the divergence clarifies when each is right. An integrated driver is the better answer when dots must move independently and quickly. It is the worse answer when the mechanism is bistable and the refresh is deliberately serial. The mechanism decides the driver, not the other way around.

---

## Key Takeaways

- The textbook answer to a component-level requirement can be the wrong answer at system scale. An H-bridge is right for one reversible load and wrong for forty.
- A better-engineered part can still be the wrong part when its strengths address a problem the system does not have. Over-specification is a real failure mode, not a safe default.
- Each iteration died to a *different* criterion — component count, then capability mismatch. Neither would have been visible from the other's analysis.
- Prefer a topology where the fault cannot be expressed over one where it must be prevented. This is the same lesson as the v1 enable pull-downs, applied to architecture selection rather than to a pin.
- Arriving independently at a published topology, after exhausting the alternatives, produced a much better understanding of *why* that topology is correct than adopting it first would have.
