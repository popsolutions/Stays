<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# innerjib7ea-rev-a — POPC_16A first-silicon FPGA validation board

## Status

**Skeleton landed.** Empty-but-valid KiCad 8 project files are
now in this directory:

- `innerjib7ea.kicad_pro` — project envelope (JSON, KiCad 8 schema).
- `innerjib7ea.kicad_sch` — empty root schematic (sexpr, version
  20231120 / KiCad 8.0).
- `innerjib7ea.kicad_pcb` — empty board (sexpr, version 20240108 /
  KiCad 8.0), 2-layer copper stack with the canonical KiCad 8
  user-layer enumeration; net 0 declared per spec; a placeholder
  100 mm × 100 mm square outline on `Edge.Cuts` (KiCad 8 DRC
  rejects boards with no Edge.Cuts edges — the placeholder
  satisfies that rule). The real mini-ITX 170 mm × 170 mm
  envelope replaces it in roadmap item #6 (Layout pass).
- `fp-lib-table` / `sym-lib-table` — empty per-project library
  tables (version 7) so the project is library-isolated and
  contributors don't depend on each other's global KiCad config.

The schematic and PCB are intentionally empty: the goal of this
PR is a project KiCad 8 opens cleanly and that the
`kicad-erc-drc` CI job reports clean against. Schematic capture,
DDR3 SO-DIMM connector, inter-card connector, layer stackup, and
layout fill the remaining roadmap items below in subsequent PRs.

## Spec (locked by ADR-001, with 2026-05-05 amendments)

| Item | Value |
|------|-------|
| FPGA | Lattice ECP5-85F (LFE5UM5G-85F-8BG756I) |
| Memory | **DDR3L-1600 SO-DIMM (1.35 V, `SSTL135`)**, single channel, **acoplável**. Reference SKU: Crucial CT4G3S160BM (4 GB, 1.35 V). See [`../../docs/hw/ddr3l-decision.md`](../../docs/hw/ddr3l-decision.md) for why DDR3L (not standard DDR3 1.5 V). |
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

1. **KiCad project skeleton.** ✅ Landed — see Status above.
   Files were authored against the published KiCad 8 file-format
   spec (sexpr-pcb / sexpr-schematic / fp-lib-table /
   sym-lib-table) because `kicad-cli` is not installed on the
   contributor workstation; the CI `kicad-erc-drc` job is the
   round-trip authority and runs `kicad-cli sch erc` and
   `kicad-cli pcb drc` on the PR.
2. **Schematic capture.** Multi-week effort split into one PR per
   schematic page for reviewability. Day-1 planning artefacts (the
   contract that the placement PRs honour) live in:
   - [`../../docs/hw/schematic-page-breakdown.md`](../../docs/hw/schematic-page-breakdown.md)
     — the eight pages (P1 block diagram, P2 ECP5-85F core, P3
     DDR3L SO-DIMM, P4 GbE host link, P5 inter-card connector, P6
     USB-UART debug + JTAG, P7 power tree, P8 boot flash) and what
     each page owns.
   - [`../../docs/hw/symbol-library-inventory.md`](../../docs/hw/symbol-library-inventory.md)
     — every symbol by source: KiCad std libs vs vendor (Lattice
     / SnapEDA / Ultra-Librarian) vs hand-authored.
   - [`../../docs/hw/decoupling-topology.md`](../../docs/hw/decoupling-topology.md)
     — per-rail decoupling spec for the ECP5-85F per FPGA-TN-02038
     and FPGA-TN-02206 (bulk + bypass + SerDes-specific filter).
     Topology only; exact counts deferred to placement.

   Power tree summary (12 V → 3.3 V / 2.5 V / 1.8 V / 1.35 V /
   1.20 V / 1.0 V / 0.675 V, including the 2.5 V analog and 1.0 V
   digital rails for the SGMII PHY); GbE chain (LiteEth → ECP5
   SerDes → SGMII PHY → magnetics → RJ45); JTAG header; USB-UART
   bridge. Day-1 planning selects **Marvell 88E1512** as the
   primary SGMII PHY (KSZ9031RNX is the documented substitute) and
   **FT232HL** as the primary USB-UART bridge (CP2102N is the
   documented cost-down substitute). **Note:** the 1.5 V DDR3 main
   rail and 0.75 V VTT termination (which earlier drafts of this
   list assumed) are removed — see
   [`../../docs/hw/ddr3l-decision.md`](../../docs/hw/ddr3l-decision.md);
   rev-A is DDR3L only, so the module rail is 1.35 V and VTT is
   0.675 V (VDD/2).
3. **DDR3L SO-DIMM connector** + routing-constraint annotation
   (50 Ω single-ended, 100 Ω differential, length-match groups).
   IO standard `SSTL135_I` / `SSTL135D_I` per the ECP5 IO bank
   capability (the chip cannot drive `SSTL15` standard-DDR3 1.5 V
   without an external level shifter — see
   [`../../docs/hw/ddr3l-decision.md`](../../docs/hw/ddr3l-decision.md)).
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
