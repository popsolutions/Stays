<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# rev-A schematic symbol library inventory

**Status:** Day-1 planning (2026-05-06). Companion to
[`schematic-page-breakdown.md`](schematic-page-breakdown.md).

> **Scope.** Catalogue every symbol the rev-A schematic needs and
> name where each one comes from: KiCad 8 standard libraries, a
> vendor portal (Lattice / SnapEDA / Ultra-Librarian), or
> hand-authored. The placement PRs (#6c–#6i) source their symbols
> from this inventory. No symbols are checked into the project
> library at this stage — this doc is a manifest, not a binary blob.

## Library strategy

The KiCad project under `kicad/innerjib7ea-rev-a/` ships with
**empty per-project library tables** (`sym-lib-table` and
`fp-lib-table` at version 7) per PR #23. This is deliberate:

- The project is **library-isolated** so contributors do not need
  to align their global KiCad config to ours.
- Each placement PR adds **only the symbols and footprints it
  uses** to the per-project library, keeping the diff reviewable.
- Standard KiCad symbols (resistors, capacitors, diodes,
  connectors with KiCad-canonical footprints) are referenced from
  the user's globally-installed KiCad 8 libraries — they are not
  re-vendored into the project library, because every KiCad 8
  install ships with them.
- Vendor and hand-authored symbols **are vendored** into
  `kicad/innerjib7ea-rev-a/lib/` so the build is reproducible
  without external network access.

### Library directory layout (target, lands per PR)

```
kicad/innerjib7ea-rev-a/
├── innerjib7ea.kicad_pro
├── innerjib7ea.kicad_sch
├── innerjib7ea.kicad_pcb
├── sym-lib-table              ← updated as symbols land
├── fp-lib-table               ← updated as footprints land
└── lib/
    ├── innerjib7ea.kicad_sym  ← hand-authored symbols
    ├── innerjib7ea.pretty/    ← hand-authored footprints
    ├── vendor-lattice.kicad_sym
    ├── vendor-marvell.kicad_sym
    ├── vendor-snapeda.kicad_sym
    ├── vendor-ultralibrarian.kicad_sym
    └── vendor-*.pretty/       ← matching vendor footprints
```

Vendor library files are split by source so any one vendor's licence
or terms-of-use can be honoured independently.

## License compatibility (vendor symbol provenance)

| Vendor source | Typical licence | Compatible with CERN-OHL-S v2 hardware? | Notes |
|---|---|---|---|
| KiCad standard libraries | CC-BY-SA 4.0 | yes | distributed with KiCad |
| Lattice (direct from vendor) | varies — usually free-to-use, no redistribution clause | yes for the ECP5 page once we re-author the symbol from the published pinout in the datasheet | Pinout is factual, not copyrightable; we author the symbol ourselves and cite the datasheet |
| SnapEDA | SnapEDA EULA — free for commercial use, attribution required, redistribution allowed | yes with attribution | each symbol's provenance recorded inline in the symbol's `Description` and `Footprint` filter fields |
| Ultra-Librarian | similar to SnapEDA — free with attribution | yes with attribution | same |
| Hand-authored | CERN-OHL-S v2 (project licence) | yes | author this only when no vendor symbol exists |

The cooperative's hardware licence is **CERN-OHL-S v2** per
`PCB_DESIGN.md`. That is the strongly-reciprocal copyleft licence
for hardware. Vendor symbols whose licence forbids redistribution
are **not** acceptable; this inventory blocks any such symbol with
an "avoid" tag below.

## Symbol inventory by page

### P1 — Block diagram

Hierarchical sheet symbols are KiCad primitives — no external
symbol library needed.

### P2 — ECP5-85F core

| Ref | Part | Source | Status |
|---|---|---|---|
| U1 | LFE5UM5G-85F-8BG756I (Lattice ECP5-5G, 756-ball BGA) | **Hand-authored** | author from FPGA-DS-02012-3.3 §4 (Pinout). Lattice publishes a Symbol Generator service for some KiCad-compatible formats but coverage of ECP5-5G in the BG756 package is incomplete; safer to hand-author and split by power domain. |
| | Bypass / decoupling caps (0.1 µF, 1 nF, 4.7 µF, 10 µF, 22 µF) | KiCad std (`Device:C`) | use `C_Small` for schematic clarity |
| | Bulk caps (47 µF, 100 µF aluminium polymer) | KiCad std (`Device:CP`) | polarised |
| | Pull-up / pull-down resistors (10 kΩ, 4.7 kΩ, 1 kΩ) | KiCad std (`Device:R`) | |
| | Configuration mode straps (DIP switch or zero-Ω jumpers) | KiCad std (`Switch:SW_DIP_x04`) | one strap bank per CFG[2:0] line |

**Decision.** Hand-author the ECP5 symbol rather than pull from a
vendor portal. Split by power domain (one symbol unit per VCCIO
bank, one for VCC core, one for VCCAUX, one for SerDes, one for
GND, one for IO) following the OrangeCrab / Versa-ECP5
convention. This split makes each schematic page easier to read
and matches how the placement PR P2 partitions the work.

### P3 — DDR3L SO-DIMM

| Ref | Part | Source | Status |
|---|---|---|---|
| J1 | 204-pin SO-DIMM connector (Molex 78171 / TE 1473005) | **SnapEDA** (preferred) or hand-author | SnapEDA hosts both Molex 78171-2042 and TE 1473005-1; either is acceptable. Footprint mechanical drawing is in JEDEC MO-244 — we cross-check the SnapEDA footprint against MO-244 before merge. |
| | VTT regulator (TPS51200 or equivalent) | **SnapEDA** | TI publishes KiCad symbols on its product pages via Ultra-Librarian; SnapEDA mirrors them. |
| | Series termination Rs (33 Ω 0402 / 0603) | KiCad std (`Device:R`) | |
| | Decoupling caps for VTT, VREF | KiCad std (`Device:C`) | |
| | I²C pull-ups for SPD bus | KiCad std (`Device:R`) | 4.7 kΩ |

### P4 — GbE host link

| Ref | Part | Source | Status |
|---|---|---|---|
| U2 | Marvell Alaska 88E1512 SGMII PHY (32-pin QFN) | **Hand-authored** (preferred) or SnapEDA | the public 88E151x_PB does not include a vendor KiCad symbol; SnapEDA has community-uploaded variants of varying quality. Hand-author from the public-brief pinout for safety. |
| T1 | RJ45 with integrated magnetics (HanRun HR911105A) | **SnapEDA** | well-stocked symbol on SnapEDA; verify the magnetics tap topology against HanRun datasheet before merge. |
| J2 | RJ45 jack with integrated LEDs | bundled into T1 | |
| | AC-coupling caps (0.1 µF, 0402 X7R) | KiCad std (`Device:C`) | series in SGMII pair |
| | Termination Rs for SGMII pair | KiCad std (`Device:R`) | |
| | Bob Smith network (75 Ω + 1 kV cap) | KiCad std (`Device:R`, `Device:C`) | EMI termination on RJ45 unused pins |
| | 25 MHz crystal for PHY | **SnapEDA** | crystal symbol straight from SnapEDA; verify load-cap rating |
| | 22 pF crystal load caps | KiCad std (`Device:C`) | NP0 dielectric required |

> **Substitute.** If SnapEDA's 88E1512 symbol is acceptable upon
> cross-check at placement time (verify all 32 pins against
> 88E151x_PB, verify package footprint against datasheet land
> pattern), it can replace the hand-authored entry. Provenance
> field in the symbol records the decision.

### P5 — Inter-card connector

| Ref | Part | Source | Status |
|---|---|---|---|
| J3 | Hirose FX18-40P-0.8SH (40-pin 0.8 mm pitch board-to-board) | **Hirose** (vendor portal) → typically distributed via SnapEDA / Ultra-Librarian | Hirose publishes 3D STEP and PCB land patterns on its product page; SnapEDA mirrors KiCad libraries for FX18-series connectors. Pull from SnapEDA, cross-check against Hirose's published land pattern. |
| | AC-coupling caps for MAST diff pairs | KiCad std (`Device:C`) | 0.1 µF X7R |
| | Pull-ups for sideband signals (`MAST_PRESENT#`, `MAST_INT#`, `MAST_RESET#`) | KiCad std (`Device:R`) | 10 kΩ |
| | I²C pull-ups for `MAST_I2C_*` | KiCad std (`Device:R`) | 4.7 kΩ |

### P6 — USB-UART debug + JTAG

| Ref | Part | Source | Status |
|---|---|---|---|
| U3 | FTDI FT232HL (LQFP-48) | **SnapEDA** | FTDI publishes KiCad symbols via Ultra-Librarian; SnapEDA mirrors. |
| J4 | Micro-USB B receptacle | KiCad std (`Connector:USB_B_Micro`) | KiCad ships canonical Micro-USB symbols and footprints |
| | ESD diode array on USB D±  | **SnapEDA** | TPD2E001 or USBLC6-2SC6 — both are stock SnapEDA |
| J5 | JTAG 2x5 0.05" pitch shrouded header | KiCad std (`Connector:Conn_02x05_Odd_Even`) | KiCad ships the canonical 2x5 SMD JTAG footprint |
| | FT232HL decoupling, ferrite bead | KiCad std (`Device:C`, `Device:FerriteBead`) | |

### P7 — Power tree

| Ref | Part | Source | Status |
|---|---|---|---|
| J6 | Molex KK254 4-pin power input | KiCad std (`Connector:Conn_01x04`) | KK254 footprint in `Connector_Molex.pretty`  |
| | Buck regulator ICs (placeholder — exact part deferred to placement) | **Ultra-Librarian** mirrored via TI / Analog Devices product pages | both TI and ADI publish KiCad-compatible symbols; pull-at-placement |
| | LDO for 0.675 V VTT (TPS51200 or alternative) | **SnapEDA** / **Ultra-Librarian** | shared with P3 (the VTT generator could live on either page; placement decides) |
| | Inductors (1 µH–22 µH, low-DCR power inductors) | KiCad std (`Device:L`) | exact value follows regulator selection |
| | Bulk + ceramic caps for 7 rails | KiCad std (`Device:C`, `Device:CP`) | |
| | TVS clamp (SMBJ15A or SMAJ15A) | **SnapEDA** | |
| | Power-good IC (TPS65086 or LTC2937) optional | **Ultra-Librarian** | optional component; deferred |
| | Sequencing logic (3-input AND `74LVC1G11` or PMIC-internal) | KiCad std (`74xx:74LVC1G11`) | KiCad has the 74xx family stocked |

### P8 — Boot flash

| Ref | Part | Source | Status |
|---|---|---|---|
| U4 | Cypress S25FL128SAGMFI001 (128 Mb quad-SPI NOR, SOIC-8) | **SnapEDA** | well-stocked symbol; SOIC-8 footprint is KiCad std |
| | SPI bus pull-ups | KiCad std (`Device:R`) | 10 kΩ on CS#, HOLD# |
| | Decoupling caps | KiCad std (`Device:C`) | |

## Hand-authored symbols — authoring rules

When a symbol is hand-authored (per the table above: U1 ECP5,
optionally U2 88E1512), the following rules apply, codified in
the project's symbol-template file (lands with PR #6c):

1. **Pin numbering** matches the package datasheet exactly.
2. **Pin name** matches the datasheet's pin name (do not abbreviate).
3. **Pin type** classification:
   - Power pins: `power_in`.
   - Ground pins: `power_in`, named `GND`.
   - Configuration pins: `input` if always driven, `bidirectional`
     if can be both input and output.
   - Reserved pins: `unconnected` (KiCad ERC will then complain
     loudly if anyone wires to them).
4. **Symbol field `Description`** cites the datasheet section that
   was the source: e.g. `FPGA-DS-02012-3.3 §4.2 BG756 pinout
   (2024-Q4)`.
5. **Symbol field `Datasheet`** is a stable URL (Lattice document
   number + version) so future editors can re-fetch the source.
6. **Multi-unit split**: power, IO bank, SerDes, configuration each
   on their own unit, in that order.

## Footprint inventory (paired with symbol inventory)

Every symbol above maps to one of these footprint sources:

- **KiCad std `*.pretty`** — `Resistor_SMD`, `Capacitor_SMD`,
  `Inductor_SMD`, `Connector_USB`, `Connector_PinHeader_2.54mm`,
  `Connector_Molex`, `Package_BGA`, `Package_QFP`, `Package_SO`.
- **SnapEDA-provided footprints** — paired with the symbol;
  cross-checked against the datasheet land pattern recommendation
  before merge. Ball-pad / pad-stack adjustments (for JLCPCB
  fab-class minimum trace/space) recorded inline.
- **Hand-authored** (only U1 BG756 if no vendor footprint passes
  cross-check) — author from the IPC-7351B nominal land pattern
  using the BG756 package mechanical drawing in FPGA-DS-02012-3.3
  §3.5.

## Cross-checks before any vendor symbol is merged

Each placement PR that pulls a SnapEDA / Ultra-Librarian symbol
must run these checks in the PR description:

1. **Pin count + pin order** — visually compare every pin against
   the published datasheet pinout.
2. **Pin type** — check power, ground, NC, IO, OD pins are typed
   correctly so KiCad ERC catches violations.
3. **Footprint land pattern** — overlay the imported footprint on
   the datasheet's recommended land pattern and confirm to within
   IPC-7351B Class B (general) or Class C (high-reliability)
   tolerance.
4. **3D model** — if SnapEDA ships a STEP file, render it in KiCad
   3D viewer and confirm height + orientation match datasheet.
5. **License attribution** — if SnapEDA / Ultra-Librarian license
   requires attribution, record it in the PR description and the
   symbol's `Description` field.

## What this doc deliberately does **not** do

- **Vendor any symbols.** All symbol files land in the placement
  PRs that use them. This doc is a manifest only.
- **Pick exact regulator parts.** P7's placement PR authors the
  regulator decision matrix.
- **Author the ECP5 symbol.** That happens in PR #6c when the FPGA
  page is placed. This doc commits to the *strategy* of
  hand-authoring it.
- **Mandate a specific SnapEDA account.** The cooperative's
  account is one option; any contributor may pull symbols under
  their own account as long as the SnapEDA EULA's attribution
  requirement is honoured.

## References

- ADR-001 — [`../adr/0001-fpga-target.md`](../adr/0001-fpga-target.md)
- Schematic page breakdown —
  [`schematic-page-breakdown.md`](schematic-page-breakdown.md)
- Decoupling topology —
  [`decoupling-topology.md`](decoupling-topology.md)
- DDR3L decision — [`ddr3l-decision.md`](ddr3l-decision.md)
- KiCad 8 library file format — `sym-lib-table` v7 spec.
- IPC-7351B — Generic Requirements for Surface Mount Design and
  Land Pattern Standard.
- JEDEC MO-244 — DDR3 / DDR3L SO-DIMM mechanical / pinout.
- Lattice **FPGA-DS-02012-3.3** — ECP5 / ECP5-5G Family Data Sheet.
- SnapEDA EULA — https://www.snapeda.com/about/terms/ (attribution
  + redistribution terms — checked at every symbol pull).
- Ultra-Librarian terms-of-use — https://www.ultralibrarian.com/.
- CERN-OHL-S v2 — project hardware licence.

Authored by Agent 2 (FPGA Hardware).
