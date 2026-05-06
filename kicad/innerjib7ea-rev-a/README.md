<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# innerjib7ea-rev-a — POPC_16A first-silicon FPGA validation board

## Status

**Skeleton.** KiCad project files not yet committed (no
`.kicad_pro` in this directory yet). This README precedes the
KiCad project to (a) document the intended scope, (b) make
ADR-001's locked decision visible at the directory root, and
(c) enumerate the sequence of follow-up PRs that fill this
directory.

## Spec (locked by ADR-001, with 2026-05-05 amendments)

| Item | Value |
|------|-------|
| FPGA | Lattice ECP5-85F (LFE5UM5G-85F-8BG756I) |
| Memory | DDR3-1600 SO-DIMM, single channel, **acoplável** |
| Open toolchain | yosys + nextpnr-ecp5 + prjtrellis + LiteDRAM + LiteEth (rev A; LitePCIe in rev B) |
| Host link | GbE 1000BASE-T via LiteEth + ECP5 SerDes → SGMII PHY → RJ45 (rev A; PCIe returns in rev B) |
| Form factor | Mini-ITX SBC, 170 mm × 170 mm |
| Inter-card connector | PCIe-style edge fingers (mechanical only) for MAST-link, physically present from rev A |
| BOM target | R$ 1500–2500 / assembled board, 5–10 unit batches |
| Fab target | JLCPCB or PCBWay, controlled-impedance 8-layer |

See [`../../docs/adr/0001-fpga-target.md`](../../docs/adr/0001-fpga-target.md)
for the full decision record (rationale, trade-offs, roadmap, and
the 2026-05-05 amendment log covering the host-link + form-factor
pivot).

## Why GbE for rev A (not PCIe)

Per ADR-001's "Why GbE for rev-A" section: the LitePCIe ECP5 PHY
does not exist upstream (Agent 4's rev-A upstream survey,
`../../docs/upstream-contributions/0001-rev-a-known-upstream-issues.md`,
"Top finding") and authoring one is multi-quarter work. GbE via
LiteEth is upstream-mature on ECP5 today and technically sufficient
for the rev-A reference workload (TinyLlama-1.1B: 700 MB load in
~6 s; token streaming ≪ 1 Gbps). PCIe returns in rev B with
CertusPro-NX, where LitePCIe's `lfcpnxpciephy.py` wrapper is
already shipping and supported.

## Multi-card requirement (non-negotiable)

This PCB **must include the inter-card connector physically** even
though rev-A ships single-card. Two-card aggregation is the
post-MVP demo. Single-card-only PCBs are not acceptable as a
final form. The connector may be electrically idle on a single
card, but it must be on the board.

## Roadmap

Sequential PRs land in this directory in approximately this order,
one PR per step:

1. **KiCad project skeleton.** Coherent `.kicad_pro` +
   empty-but-valid `.kicad_sch` + empty-but-valid `.kicad_pcb`,
   produced with `kicad-cli` so files round-trip cleanly.
2. **Schematic capture.** ECP5-85F symbol; full power tree
   (12 V → 3.3 V / 2.5 V / 1.8 V / 1.5 V / 1.35 V / 1.0 V / 0.75 V,
   including the 2.5 V analog and 1.0 V digital rails for the
   SGMII PHY); GbE chain (LiteEth → ECP5 SerDes → KSZ9031RNX SGMII
   PHY → magnetics → RJ45); JTAG header; USB-UART bridge.
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
