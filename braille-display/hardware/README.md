# Hardware

PCB design files, schematics, and mechanical files.

## Contents

- `/pcb/` — Gerbers, schematic PDF, BOM export from Fusion 360
- `/mechanical/` — 3D print files if applicable

## Exporting from Fusion 360

When ready to commit PCB files:
1. Export schematic as PDF → `/hardware/pcb/schematic.pdf`
2. Export Gerbers → `/hardware/pcb/gerbers/`
3. Export BOM → `/bom/bom.csv`

Do not commit `.f3d` source files — use PDF + Gerbers for documentation.
