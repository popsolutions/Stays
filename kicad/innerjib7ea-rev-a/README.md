<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# innerjib7ea-rev-a — POPC_16A first-silicon FPGA validation board

## Status

**Skeleton.** KiCad project files not yet committed (no
`.kicad_pro` in this directory yet). This README precedes the
KiCad project to (a) document the intended scope, (b) make
ADR-001's locked decision visible at the directory root, and
(c) enumerate the sequence of follow-up PRs that fill this
directory.

## Spec (locked by ADR-001)

| Item | Value |
|------|-------|
| FPGA | Lattice ECP5-85F (LFE5UM5G-85F-8BG756I) |
| Memory | DDR3-1600 SO-DIMM, single channel, **acoplável** |
| Open toolchain | yosys + nextpnr-ecp5 + prjtrellis + LiteDRAM + LitePCIe |
| Host link | PCIe Gen2 (hard IP on ECP5) |
| Inter-card connector | Required physically present (multi-card mandate) |
| BOM target | R$ 1500–2500 / assembled board, 5–10 unit batches |
| Fab target | JLCPCB or PCBWay, controlled-impedance 8-layer |

See [`../../docs/adr/0001-fpga-target.md`](../../docs/adr/0001-fpga-target.md)
for the full decision record (rationale, trade-offs, roadmap).

## Multi-card requirement (non-negotiable)

This PCB **must include the inter-card connector physically** even
though rev-A ships single-card. Two-card aggregation is the
post-MVP demo. Single-card-only PCBs are not acceptable as a
final form. The connector may be electrically idle on a single
card, but it must be on the board.

## Roadmap

Sequential PRs land in this directory in approximately this order
(one PR per step, per `feedback_solo_mode_and_pr_workflow`):

1. **KiCad project skeleton.** Coherent `.kicad_pro` +
   empty-but-valid `.kicad_sch` + empty-but-valid `.kicad_pcb`,
   produced with `kicad-cli` so files round-trip cleanly.
2. **Schematic capture.** ECP5-85F symbol, power tree
   (12 V → 3.3 V / 1.8 V / 1.5 V / 1.35 V / 0.75 V), JTAG
   header, USB-UART bridge.
3. **DDR3 SO-DIMM connector** + routing-constraint annotation
   (50 Ω single-ended, 100 Ω differential, length-match groups).
4. **Inter-card connector** (per multi-card requirement).
5. **Layer stackup** planning (8-layer for DDR3 controlled
   impedance — see `docs/PCB_DESIGN.md` for the target stackup).
6. **Layout pass.**
7. **ERC + DRC clean** (CI `kicad-erc-drc` job green on the diff).
8. **Gerbers, BOM, position files** for JLCPCB.

## Contributors

Hardware authoring in `kicad/` is owned by Agent 2 in the
PopSolutions Sails 4+1 operating model. Cross-stream changes
require a `cross-stream` issue.
