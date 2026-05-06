<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# rev-A SO-DIMM voltage class — DDR3L (1.35 V, SSTL135)

**Status:** Locked (2026-05-06). Closes Stays issue #27.

> **TL;DR.** rev-A's SO-DIMM is **DDR3L** (1.35 V, `SSTL135`). It is
> **not** standard DDR3 (1.5 V, `SSTL15`). The Lattice ECP5 IO banks
> physically cannot drive `SSTL15` without an external level
> shifter. Every BOM line item, every schematic annotation, every
> rail in the power tree, and every IO-bank assignment for the
> SO-DIMM byte lanes follows from this single fact. Substituting a
> 1.5 V DDR3 SO-DIMM into a working rev-A board will not function
> and may damage the FPGA IO banks.

## Why DDR3L

### ECP5 IO bank capability (the single hard constraint)

The Lattice ECP5 family's left and right IO banks (the only banks
suitable for high-speed DRAM byte lanes) support the following SSTL
class IO standards natively:

- `SSTL135_I` / `SSTL135_II` — single-ended, 1.35 V (DDR3L)
- `SSTL135D_I` / `SSTL135D_II` — differential, 1.35 V (DDR3L
  DQS / CK pairs)

They do **not** support `SSTL15` (1.5 V standard-DDR3) on the
banks used for DRAM. This is documented in the ECP5 family
datasheet (Lattice doc DS1044) and codified in every open-source
ECP5 reference design that uses DDR3:

| Reference design | DDR3 module | IO standard | Voltage class |
|---|---|---|---|
| OrangeCrab (LFE5U-25F / 85F) | Micron MT41K128M16 | `SSTL135_I` | DDR3L |
| Lattice Versa-ECP5 (LFE5UM5G) | Micron MT41K64M16 | `SSTL135_I` | DDR3L |
| Trellis Board (LFE5UM5G-85F) | Micron MT41J256M16 | `SSTL135_I` | **standard DDR3 module under-volted** |

The Trellis Board case (a standard-DDR3 1.5 V `MT41J*` module wired
to the ECP5's `SSTL135` IO standard) works **only** because the
Trellis Board has dedicated voltage regulation that under-volts the
DRAM module to 1.35 V. That is a one-off bench-oriented hack — it
is not the right path for a production rev-A board with a
user-replaceable SO-DIMM, because:

- A user could replace the SO-DIMM with a stock 1.5 V part and
  expect it to work (it would not, and the under-volt path
  forecloses the option of supplying 1.5 V at all).
- Under-volting voids the JEDEC timing guarantees on standard DDR3
  parts.
- The "user can upgrade RAM" thesis from ADR-001 only holds if the
  socketed module's voltage class is the standard one (DDR3L
  modules are interchangeable; no vendor ships a "Trellis-special
  under-volted DDR3" SKU).

DDR3L SO-DIMMs are commodity parts at retail and are
interchangeable with each other. Choosing DDR3L from rev-A makes
the cooperative's "user upgradeable" thesis hold.

### Confirmed by Agent 4's day-1 LiteDRAM recon

The 2026-05-06 LiteDRAM ECP5 PHY recon
([`../upstream-contributions/2026-05-06-litedram-ecp5.md`](../upstream-contributions/2026-05-06-litedram-ecp5.md))
section *3. IO standard compatibility* already states this finding
verbatim, citing the relevant snippet from
`litex_boards/platforms/lattice_versa_ecp5.py`:

```python
("ddram", 0,
    Subsignal("a",     Pins("..."), IOStandard("SSTL135_I")),
    Subsignal("ba",    Pins("..."), IOStandard("SSTL135_I")),
    Subsignal("dq",    Pins("..."), IOStandard("SSTL135_I")),
    Subsignal("dqs_p", Pins("..."), IOStandard("SSTL135D_I")),
    Subsignal("clk_p", Pins("..."), IOStandard("SSTL135D_I")),
    ...
```

The recon doc concludes: *"ECP5's IO banks **cannot** drive 1.5V
SSTL15 DDR3 directly without an external level-shifter, so DDR3L is
the correct path."* This document locks that conclusion into the
rev-A spec.

### Mechanical compatibility

The SO-DIMM mechanical envelope (204-pin DDR3 / DDR3L) is
**identical** between standard DDR3 and DDR3L. The same connector
(Molex 78171 series, TE 1473005, etc.) accepts both module classes
— the difference is electrical (1.5 V vs 1.35 V module VDD, the
SPD bytes that declare voltage support, and the `SSTL15` vs
`SSTL135` IO levels). This means **the rev-A schematic cannot
distinguish DDR3 from DDR3L at the connector symbol** — the
distinction lives in:

1. The power rail feeding the connector's `VDD` pins (1.35 V).
2. The `VTT` termination level (~0.675 V = VDD/2).
3. The IO standard assigned to the FPGA pins driving the byte
   lanes (`SSTL135_I` / `SSTL135D_I`).
4. **The BOM line item for the reference DIMM SKU** (DDR3L only).

Therefore the documentation in this file and in the
[`PCB_DESIGN.md`](../PCB_DESIGN.md) rev-A subsection is the
authoritative record. Schematic capture (Stays #6) and the SO-DIMM
connector PR (Stays #7) must read this document and propagate the
constraint into the schematic + BOM.

## Reference DIMM SKU

**Reference SKU for rev-A bring-up: Crucial CT4G3S160BM.**

Specs:

- **Class:** DDR3L (low-voltage), 1.35 V
- **Speed:** PC3L-12800 (DDR3L-1600, 1600 MT/s)
- **Capacity:** 4 GB
- **Form factor:** 204-pin SO-DIMM, unbuffered, non-ECC
- **Rank:** Single-rank (1Rx8)
- **JEDEC profile:** Standard SPD; advertises 1.35 V support per
  JEDEC SPD byte 11 (low-voltage capable). Backward-compatible
  1.5 V operation is also declared in the SPD on most DDR3L parts,
  which means a vendor's BIOS/firmware can choose either rail;
  rev-A always drives 1.35 V.
- **Sourcing:** Mouser, Digi-Key, JLCPCB component library at the
  time of writing. Crucial is the Micron consumer brand, so the
  underlying die is one of the `MT41K*` parts that the
  open-FPGA + LiteDRAM ecosystem has been validating against
  since 2019 (OrangeCrab, Versa-ECP5).
- **Why this SKU specifically:** It is among the smallest /
  lowest-cost DDR3L-1600 SO-DIMMs in volume retail at the rev-A
  reference workload size (TinyLlama-1.1B = ~700 MB weight set;
  4 GB is comfortable; larger capacities are pure cost overhead
  for rev-A).

### Acceptable substitutes (any DDR3L-1600 unbuffered non-ECC SO-DIMM)

The cooperative's procurement (Marcos's responsibility, per the
solo-mode operating model) may substitute any DDR3L-1600 SO-DIMM
that meets these criteria:

- 1.35 V module VDD (DDR3L, **not** DDR3)
- 204-pin SO-DIMM mechanical
- Unbuffered, non-ECC
- 4–8 GB capacity (single- or dual-rank both acceptable)
- DDR3L-1333 or DDR3L-1600 speed grade (the LiteDRAM ECP5 PHY
  caps the practical clock at ~800 MT/s on ECP5 anyway, per the
  recon doc; the speed grade above that is headroom)
- Vendor with a published JEDEC SPD profile (Crucial / Micron,
  Kingston, Samsung, SK Hynix, ADATA, etc.)

Examples that meet these criteria:

| Vendor | SKU | Capacity | Notes |
|---|---|---|---|
| Crucial | CT4G3S160BM | 4 GB | reference (above) |
| Crucial | CT8G3S160BM | 8 GB | dual-rank substitute |
| Kingston | KVR16LS11/4 | 4 GB | DDR3L-1600 single-rank |
| Samsung | M471B5273DH0-YK0 | 4 GB | DDR3L-1600 |

### What you must not substitute

- **Standard DDR3 1.5 V SO-DIMMs** (e.g. `MT41J*`-based parts).
  They will not negotiate a 1.35 V rail and ECP5 cannot drive them
  at 1.5 V; the board will either not POST or stress the IO banks
  out of spec over time.
- **DDR3L RDIMMs** (registered) — rev-A is unbuffered.
- **DDR3L UDIMMs** (full-size desktop DIMM) — wrong form factor.
- **DDR4 SO-DIMMs** (260-pin, 1.2 V) — wrong electrical class
  *and* wrong mechanical key. Mechanically blocked anyway.

## What changes downstream of this decision

| Doc / artefact | Pre-decision | Post-decision |
|---|---|---|
| `docs/PCB_DESIGN.md` connector table | "unbuffered, 1.5 V" | "unbuffered, DDR3L 1.35 V SSTL135" + reference SKU |
| `docs/PCB_DESIGN.md` power tree | separate 1.5 V DDR3 + 1.35 V FPGA core rails | single 1.35 V rail feeds DDR3L module + ECP5 SSTL135 IO bank |
| `docs/PCB_DESIGN.md` VTT rail | 0.75 V (= 1.5 V / 2) | 0.675 V (= 1.35 V / 2) |
| `docs/PCB_DESIGN.md` byte lane IO standard | unspecified | `SSTL135_I` / `SSTL135D_I` |
| `kicad/innerjib7ea-rev-a/README.md` spec table | "DDR3-1600 SO-DIMM" | "DDR3L-1600 SO-DIMM (1.35 V, `SSTL135`)" + reference SKU |
| Stays #6 (schematic capture) power-tree call-outs | 1.5 V DDR3 main rail | 1.35 V DDR3L+VCCIO rail |
| Stays #7 (SO-DIMM connector PR) | "DDR3 SO-DIMM" | "DDR3L SO-DIMM" + IO standard `SSTL135_I` |
| Stays #9 (8-layer stackup) power-plane split | 1.5 V / 1.35 V split | 1.35 V single rail (the 1.5 V split disappears) |

Cross-stream tracking comments will land on Stays #6, #7, and #9
referencing this document so the schematic and layout PRs inherit
the corrected spec.

## ADR-001 alignment

ADR-001 (`docs/adr/0001-fpga-target.md`) currently says only
"DDR3 SO-DIMM (acoplável), single channel" without specifying
DDR3 vs DDR3L. The amendment trail in ADR-001 already covers two
2026-05-05 amendments (host link PCIe→GbE, form factor M.2→
Mini-ITX); this DDR3L clarification is the third amendment in
spirit but should be filed against the cross-fleet ADR canonical
copy (MAST issue #32 per the rev-A amendment-tracking convention)
rather than re-amending the Stays copy independently. This
document is the rev-A-side authoritative record until that MAST
amendment lands.

## Out of scope for this document

- **Sourcing the actual DIMM** — that is BOM procurement, owned
  by Marcos under the solo-mode operating model.
- **Schematic capture itself** — Stays #6.
- **Picking the connector vendor / part number** — Stays #7.
- **Layer stackup / power-plane geometry** — Stays #9. The
  voltage-class decision narrows the rail count by one but the
  stackup is otherwise unaffected.
- **DDR3L → DDR4 → DDR5 → DDR6 trajectory** — owned by
  ADR-001's roadmap section ("Rev B / Rev C / Beyond"); this
  document only locks rev-A.

## References

- ADR-001 — [`../adr/0001-fpga-target.md`](../adr/0001-fpga-target.md)
- LiteDRAM ECP5 day-1 recon —
  [`../upstream-contributions/2026-05-06-litedram-ecp5.md`](../upstream-contributions/2026-05-06-litedram-ecp5.md)
  (sections 2 and 3 in particular)
- Lattice ECP5 family datasheet (DS1044) — IO standards table.
- ECP5 IO bank IO-standard list as exposed by `nextpnr-ecp5` and
  `prjtrellis` — `SSTL135_I`, `SSTL135_II`, `SSTL135D_I`,
  `SSTL135D_II` are the supported set on the left/right banks
  used for DRAM byte lanes.
- JEDEC JESD79-3 (DDR3) and JESD79-3-1 (DDR3L addendum) — module
  voltage class definitions.
- Crucial CT4G3S160BM product page (vendor SPD profile reference).
- Stays issue #27 — *"[cross-stream][hw] Verify rev-A SO-DIMM is
  DDR3L (1.35V SSTL135), not standard DDR3 (1.5V)"* — this
  document closes that issue.

Authored by Agent 2 (FPGA Hardware).
