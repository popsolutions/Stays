<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# PCB design process notes

This document is the running notebook for the custom PCB design work.
Each PCB rev gets its own subsection; learning carries forward.

## Common conventions

- **CAD tool**: KiCad 8 (or later when stable).
- **Fab target**: JLCPCB or PCBWay (8-12 layer capable, controlled
  impedance).
- **Manufacturing target**: small batches (5-10 boards) for prototyping.
- **License of design files**: CERN-OHL-S v2 (strongly reciprocal).
- **Naming convention**: \`kicad/<sail-codename>-rev-<letter>/\` (e.g.,
  \`kicad/innerjib7ea-rev-a/\`).

## InnerJib7EA rev A — Lattice ECP5-85F + DDR3 SO-DIMM

(forthcoming — first KiCad commit pending; see \`docs/adr/0001-fpga-target.md\`
for the locked spec.)

### Layer stackup target

8-layer stackup for DDR3 routing:
1. Top signal (high-speed)
2. Ground plane
3. Inner signal 1
4. Power plane (3.3V / 1.8V split)
5. Power plane (1.5V / 1.35V split)
6. Inner signal 2
7. Ground plane
8. Bottom signal

### Routing rules

- DDR3 lanes: controlled 50Ω single-ended / 100Ω diff, length matched
  ±5 mil within byte lane, ±10 mil across address group.
- High-speed PCIe / inter-card lanes: 100Ω diff, length matched ±2 mil
  within pair.

### Power tree

- 12V input from M.2 / molex
- 3.3V step-down (FPGA I/O)
- 1.8V step-down (DDR3 controller logic)
- 1.5V step-down (DDR3 main rail)
- 1.35V LDO (FPGA core)
- 0.75V LDO (DDR3 VTT termination)

(Topology specifics land in the schematic.)

### Bring-up checklist

(Filled in by the bring-up notebook.)

## Future revs

- rev B: ECP5 + DDR3 + inter-card connector validated (after rev A
  bring-up surfaces the inter-card protocol decisions)
- rev C: CertusPro-NX + DDR4 (when Project Oxide DDR4 PHY matures)
