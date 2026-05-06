<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# rev-A decoupling topology — ECP5-85F per FPGA-TN-02038 / FPGA-TN-02206

**Status:** Day-1 planning (2026-05-06). Companion to
[`schematic-page-breakdown.md`](schematic-page-breakdown.md) and
[`symbol-library-inventory.md`](symbol-library-inventory.md).

> **Scope.** Per-rail decoupling **topology** for the ECP5-85F
> (LFE5UM5G-85F-8BG756I). What capacitor classes go on which rail,
> in what tier (bulk / mid / bypass / high-frequency), with the
> rationale tied to Lattice's hardware checklist (FPGA-TN-02038)
> and SerDes / PCS usage guide (FPGA-TN-02206). **Exact counts and
> exact placement geometry are deferred to placement PR P2.** This
> doc is the spec; the placement PR is the implementation.

## What "topology only" means here

For each FPGA power rail, this document specifies:

1. **The capacitor tiers** to provide (bulk / mid-frequency /
   bypass / high-frequency).
2. **The capacitance values per tier** (e.g. 22 µF bulk + 4.7 µF
   mid + 100 nF bypass).
3. **The dielectric class** required (X7R / X5R / NP0 / aluminium
   polymer).
4. **The placement priority** within the BGA fanout (innermost /
   outermost ring relative to the BGA balls).

This document does **not** specify:

1. The exact **count** of bypass caps per rail (deferred to
   placement so the actual ball count and pour geometry can be
   measured).
2. The exact **footprint** size (0201 / 0402 / 0603 — deferred to
   placement; depends on JLCPCB minimum-feature class chosen).
3. The exact **part number** (ESL / ESR figures available from
   datasheets at placement time will drive the choice; same
   capacitor class can be sourced from many vendors).
4. The **PDN simulation results** — analytical SPICE / S-parameter
   verification of the PDN impedance is a separate post-layout
   activity, not part of schematic capture.

## Lattice reference documents that bound this spec

| Doc | Title | What it constrains |
|---|---|---|
| **FPGA-DS-02012-3.3** | ECP5 / ECP5-5G Family Data Sheet | Recommended operating voltages, max ripple, supply current per rail (Table 3.2). |
| **FPGA-TN-02038** | ECP5 Hardware Checklist | Decoupling capacitance values, tier counts per VCC / VCCAUX / VCCIO, ground / power return path rules. **Primary reference for the bulk / mid / bypass tier topology below.** |
| **FPGA-TN-02206** | ECP5 SerDes / PCS Usage Guide | Filtering of `VCCA`, `VCCHTX`, `VCCHRX` (the SerDes power rails). Topology differs from generic VCC because SerDes is more sensitive to phase noise. **Primary reference for the SerDes-specific filtering below.** |
| **FPGA-TN-02018** | ECP5 sysCONFIG Usage Guide | Configuration-pin power-up requirements; sets the lower bound on the decoupling for the configuration domain (typically shares VCCAUX). |

Where this document cites a tier value, it follows TN-02038 unless
explicitly overridden by TN-02206 for a SerDes rail.

## Per-rail topology

The ECP5-5G has the following power domains (FPGA-DS-02012-3.3
§3.2 Table 3.2). For each one this section names the rail
voltage, the source rail in the rev-A power tree, the decoupling
tier set, and the placement priority.

### `VCC` — FPGA core (1.20 V)

- **Source.** 1.20 V buck on power-tree page P7 (shared with
  SerDes `VCCA` / `VCCHTX` per `PCB_DESIGN.md` power tree, with
  the SerDes filter network described in
  [`#vcca-and-vcchtx--serdes-power-12v`](#vcca-and-vcchtx--serdes-power-12-v) below).
- **Tier set:**
  - **Bulk** — 1× 47 µF or 100 µF aluminium polymer, near the
    regulator output, before the inductor de-Q if any.
  - **Mid** — 4× 22 µF X5R MLCC at the BGA core perimeter
    (rough count; deferred to placement).
  - **Bypass** — 1× 4.7 µF X5R MLCC and 1× 100 nF X7R MLCC per
    `VCC` ball cluster in the BGA fanout.
  - **High-frequency** — 1× 1 nF X7R or 100 pF NP0 MLCC adjacent
    to the cluster bypass when the cluster sits on a high-activity
    bank (deferred to placement; per-cluster decision).
- **Placement priority.** 100 nF bypass innermost (closest ball
  fanout); 4.7 µF behind it; 22 µF at the BGA perimeter; bulk at
  the regulator. This is the ascending-frequency tier ladder
  TN-02038 §3.2 prescribes.
- **Dielectric.** X7R for sub-µF bypass, X5R for ≥ 4.7 µF mid /
  bulk MLCCs, aluminium polymer for the regulator-output bulk.
- **Notes.** Core current at 85F worst case is sub-1 A typical
  for inference workloads and ~1.5 A peak under heavy SerDes
  + DSP block use; the bulk + mid + bypass ladder above is
  conservative for that load.

### `VCCAUX` and `VCCAUXA` — auxiliary (2.5 V)

- **Source.** 2.5 V buck on P7. `VCCAUX` and `VCCAUXA` are tied
  per FPGA-DS-02012-3.3 Table 3.2 footnote 2.
- **Tier set:**
  - **Bulk** — 1× 22 µF aluminium polymer or X5R at the regulator
    output.
  - **Bypass** — 1× 100 nF X7R MLCC per VCCAUX / VCCAUXA ball
    (count exact at placement).
- **Placement priority.** 100 nF closest to the BGA balls; bulk at
  the regulator.
- **Notes.** This rail also supplies the SGMII PHY analog domain
  (88E1512 VDDA_2V5 if KSZ9031 substituted, VDDA_2V5 likewise) —
  add a **ferrite bead + 10 µF** filter on the PHY-side trace to
  isolate PHY analog noise from FPGA VCCAUX. Filter lives on P4,
  not on P2.

### `VCCA` and `VCCHTX` — SerDes power (1.2 V)

- **Source.** 1.20 V buck on P7, **filtered separately from
  `VCC`**. Per FPGA-TN-02206 §4 (Power Filter), SerDes rails
  must be isolated from the noisy core supply by a low-resistance
  ferrite or LC filter.
- **Tier set:**
  - **Pre-filter bulk** — 1× 22 µF X5R MLCC on the trunk side
    (shared with VCC core supply rail).
  - **Filter element** — ferrite bead (BLM18AG471 class — 470 Ω
    @ 100 MHz, low DCR) **or** small LC filter (1 µH + 10 µF X5R)
    between the VCC core trunk and the SerDes branch. Choice
    deferred to placement; ferrite is standard for ECP5.
  - **Post-filter bulk** — 1× 10 µF X5R MLCC on the SerDes side
    of the filter.
  - **Bypass** — 1× 100 nF X7R MLCC per `VCCA` / `VCCHTX` ball.
  - **High-frequency** — 1× 1 nF X7R MLCC adjacent to each
    bypass.
- **Placement priority.** Filter element placed on the trunk
  trace, close to the regulator output. Post-filter bulk
  immediately after the filter. Bypass + high-frequency caps
  innermost on the BGA fanout.
- **Notes.** **TN-02206 §4 explicitly requires filter isolation
  for SerDes power.** Sharing the regulator with VCC core is
  acceptable; sharing the post-filter copper pour is not. The
  filter component must carry the SerDes channel current
  (~50 mA worst-case per active channel) without saturating its
  ferrite — verify against datasheet at placement.

### `VCCHRX` — SerDes RX bias (0.30–1.26 V)

- **Source.** Tied to the same post-filter SerDes node as `VCCA`
  / `VCCHTX` for rev-A simplicity (FPGA-TN-02206 §4 lists this
  as the reference topology for ECP5 designs that don't need
  per-channel RX bias programming). A separate `VCCHRX` LDO is
  available as a placement-time substitute if a particular link's
  margin requires it.
- **Tier set:** Same as `VCCA` / `VCCHTX` — share the post-filter
  bulk, add 1× 100 nF + 1× 1 nF bypass per `VCCHRX` ball.
- **Placement priority.** Inherits from the SerDes filter;
  `VCCHRX` balls join the post-filter pour by short trace.
- **Notes.** Per FPGA-TN-02206 §4, biasing `VCCHRX` separately
  with an LDO is only required for very-high-jitter applications
  (PCIe Gen3 and above). Rev-A SGMII at 1.25 GBaud and MAST-link
  at PCIe-Gen2 / Gen3 class are within the shared-rail tolerance.

### `VCCIO0..7` — IO bank rails (1.35 V / 2.5 V / 3.3 V depending on bank)

- **Source.** Per-bank, from the corresponding rail on P7:
  - DDR3L byte-lane banks: 1.35 V (`SSTL135`).
  - SGMII bank: 2.5 V or 3.3 V (decision on P2).
  - General-purpose IO banks: 3.3 V.
- **Tier set (per VCCIO bank):**
  - **Bulk** — 1× 10 µF X5R MLCC at the bank perimeter.
  - **Bypass** — 1× 100 nF X7R MLCC per VCCIO ball pair (count
    exact at placement; TN-02038 §3.2 recommends ≥ 1 per 4 IO
    pins as a starting point, tightened to 1 per 2 pins for
    high-toggle banks like DDR byte lanes).
  - **High-frequency** — for DDR3L banks only: 1× 1 nF X7R MLCC
    distributed every other VCCIO ball to suppress strobe-edge
    noise.
- **Placement priority.** 100 nF closest to the BGA balls; bulk
  at the bank perimeter; high-frequency caps interleaved between
  the 100 nFs on DDR banks.
- **Notes.** The 1.35 V VCCIO bank carries the DDR3L SO-DIMM byte
  lanes; per `hw/ddr3l-decision.md` this rail is shared with the
  SO-DIMM module VDD. Decoupling on the SO-DIMM side is the
  module's responsibility; decoupling on the FPGA side is per
  this spec.

## SerDes-specific filter detail (FPGA-TN-02206 §4)

```
   1.20 V buck (P7)
        │
        ├──── VCC core trunk ──── (caps as VCC topology) ──── BGA core
        │
        └──── ferrite bead ──── post-filter pour ──── BGA SerDes balls
                                       │
                                       ├──── 10 µF post-filter bulk
                                       ├──── 1× 100 nF / VCCA ball
                                       ├──── 1× 100 nF / VCCHTX ball
                                       ├──── 1× 100 nF / VCCHRX ball (shared rail)
                                       └──── 1× 1 nF HF cap / cluster
```

The filter element selection depends on the worst-case SerDes
current draw. For two-channel use (SGMII + MAST-link single lane
active per direction in rev-A) the trunk current is small; a
470 Ω @ 100 MHz ferrite (BLM18AG471SN1D class) is the default
recommendation. For four-channel use (rev-A future bring-up of all
four MAST-link lanes simultaneously), verify the ferrite DCR
saturation margin at placement.

## DDR3L-specific decoupling on the FPGA side

`hw/ddr3l-decision.md` and `PCB_DESIGN.md` lock the DDR3L IO bank
at 1.35 V (SSTL135). The decoupling topology on the FPGA side of
the DDR3L byte lanes is per the VCCIO general spec above, with
this addition:

- **VTT termination decoupling** (lives on P3, not P2) —
  4× 10 µF X5R MLCC near the VTT regulator output, plus 1×
  100 nF X7R MLCC per termination resistor pack near each
  group of 4–8 termination resistors.
- **VREF decoupling** — 1× 100 nF X7R MLCC at the VREF generator
  output (resistor divider or VTT-IC reference output).

## Bulk decoupling at the power-input connector

P7 (the power tree page) carries its own bulk decoupling at the
12 V input:

- 1× 470 µF aluminium polymer at the connector.
- 1× 100 nF X7R MLCC at the connector.
- 1× TVS clamp (SMBJ15A or SMAJ15A) for transient suppression.

Each downstream buck regulator has its own input + output
decoupling per the regulator's reference design (lands at
placement when the exact regulator is selected — the regulator
data sheet's "typical application" schematic is the source of
record for that page's decoupling).

## High-frequency-cap deferral

TN-02038 §3.2 lists 1 nF / 100 pF / 10 pF "high-frequency caps"
as an optional outermost tier. Whether to populate that tier at
all depends on:

1. The board's PDN impedance target (set by the worst-case
   simultaneous switching output count on a single bank).
2. Whether the SerDes link sees jitter margin issues during
   bring-up.

**Decision for rev-A.** Plan the topology with placeholder
positions for high-frequency caps on DDR3L banks and SerDes
clusters; populate them only if bring-up shows a measurable need.
The 0402 / 0201 footprints reserved for them carry zero-Ω
"jumper" or DNP markings on the schematic until placement
verifies population.

## What lands in the placement PR P2 (and PR P7)

The schematic capture for P2 (ECP5 page) implements:

- One bypass cap per VCC / VCCAUX / VCCIO / SerDes ball cluster
  per the topology spec above, drawn next to the ball it serves.
- Bulk caps drawn at the bank or trunk perimeter.
- Filter element (ferrite or LC) drawn on the SerDes branch trunk.
- VTT decoupling drawn on P3, not P2.

The schematic capture for P7 (power tree page) implements:

- Per-rail input + output decoupling for each buck regulator
  per the regulator's data sheet typical application schematic.
- Power-good chain for sequencing.
- Bulk filtering at the 12 V input.

## What lands in placement layout (post-schematic)

- Exact decoupling-cap **placement geometry** (distance from
  ball, orientation relative to the GND via, via-stitch density
  on the GND return path) — driven by FPGA-TN-02038 §3.3
  guidance.
- **PDN impedance simulation** — analytical SPICE / S-parameter
  verification (pi-PDN, Sigrity, or open tools like KiCad's
  upcoming PDN analyser) lives in the layout PR or a follow-up
  bring-up validation PR.

## References

- ADR-001 — [`../adr/0001-fpga-target.md`](../adr/0001-fpga-target.md)
- `PCB_DESIGN.md` power tree —
  [`../PCB_DESIGN.md`](../PCB_DESIGN.md)
- DDR3L decision — [`ddr3l-decision.md`](ddr3l-decision.md)
- Schematic page breakdown —
  [`schematic-page-breakdown.md`](schematic-page-breakdown.md)
- Symbol library inventory —
  [`symbol-library-inventory.md`](symbol-library-inventory.md)
- Lattice **FPGA-DS-02012-3.3** — ECP5 / ECP5-5G Family Data
  Sheet (Recommended Operating Conditions, Table 3.2).
- Lattice **FPGA-TN-02038** — ECP5 Hardware Checklist
  (decoupling tier counts and placement guidance).
- Lattice **FPGA-TN-02206** — ECP5 SerDes / PCS Usage Guide
  (SerDes power filter §4).
- Lattice **FPGA-TN-02018** — ECP5 sysCONFIG Usage Guide.

Authored by Agent 2 (FPGA Hardware).
