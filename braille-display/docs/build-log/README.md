# Build Log

Dated entries documenting decisions, failures, and changes throughout the project.

## Entries

| # | Entry | Topic |
|---|-------|-------|
| 01 | [Power Architecture](01_power_architecture.md) | 2S LiPo rail design, boost converter selection — **superseded by 04** |
| 02 | [v1 Failure Analysis](02_v1_failure_analysis.md) | Shoot-through root cause — shift register powerup state |
| 03 | [v2 Redesign](03_v2_redesign.md) | Decoder swap, Schottky diode selection, enable pin strategy |
| 04 | [Power Redesign](04_power_redesign_bq25616j.md) | BQ25616J single-cell architecture, power path, JEITA thermistor, pin-level design |
| 05 | [Driver Iteration History](05_driver_iteration_history.md) | H-bridge → DRV8912 → decoder, and why each was rejected |

---

## Template for new entries

Copy the block below when starting a new entry. Save as `NN_short_topic_name.md`.

```markdown
# NN — Title

**Date:** YYYY-MM  
**Type:** [Failure analysis | Design decision | Component selection | Testing results | Architecture change | Milestone]

## Overview

Brief description of what this entry covers.

## [Context / What happened / Requirements]

...

## [Analysis / Design decisions]

...

## Key Takeaways

- Bullet point 1
- Bullet point 2
```
