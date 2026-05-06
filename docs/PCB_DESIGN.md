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

## InnerJib7EA rev A — Lattice ECP5-85F + DDR3L SO-DIMM + GbE host link

> Spec is locked by [ADR-001](adr/0001-fpga-target.md), including the
> 2026-05-05 amendments on host link (PCIe → GbE) and form factor
> (M.2 → Mini-ITX), and the 2026-05-06 DDR3L (1.35 V SSTL135)
> clarification — see [`hw/ddr3l-decision.md`](hw/ddr3l-decision.md).
>
> **Voltage class for the SO-DIMM is DDR3L (1.35 V), not standard
> DDR3 (1.5 V).** ECP5 IO banks support `SSTL135` only; standard
> DDR3 (`SSTL15`) cannot be driven without an external level
> shifter. This applies to the connector pinout (same), the rail
> (1.35 V instead of 1.5 V), and the BOM (DDR3L SO-DIMM SKUs only).

### Form factor

**Mini-ITX SBC, 170 mm × 170 mm.** Standard Mini-ITX mounting-hole
pattern and standard Mini-ITX power inputs. Selected over the
originally-assumed M.2 22110 because the DDR3 SO-DIMM connector
(67.6 mm wide), the RJ45 jack with integrated magnetics, the
inter-card edge connector, JTAG, and USB-UART do not all fit on a
22-mm-wide M.2 form factor. Mini-ITX gives ~28 900 mm² of board
area vs ~2 420 mm² for M.2 22110 — about 12× the routing space.

M.2 returns in rev B as a plug-in compute module that delegates
host link to a Mini-ITX (or other) carrier board.

### Connectors

| Connector | Part suggestion | Notes |
|-----------|-----------------|-------|
| RJ45 + magnetics | HanRun HR911105A or Bel Fuse 0826-1G1T-23-F | integrated 1000BASE-T magnetics; LEDs on jack |
| DDR3L SO-DIMM (acoplável) | Molex 78171 series (204-pin) or TE 1473005 | unbuffered, **DDR3L 1.35 V SSTL135** — see [`hw/ddr3l-decision.md`](hw/ddr3l-decision.md). Reference SKU: Crucial CT4G3S160BM (4 GB DDR3L-1600 SO-DIMM, single-rank, 1.35 V) |
| JTAG | 2×5 0.05" pitch shrouded (Tigard / Black Magic Probe compatible) | per project bring-up convention |
| USB-UART | Micro-USB B (FT232H or CP2102N bridge) | tentative; revisit when bring-up scripts land |
| Inter-card MAST-link | PCIe-style edge fingers (mechanical only) | not host PCIe; carries MAST-link diff pairs for card-to-card aggregation |
| Power input | 4-pin Molex KK254 + standard ATX 12 V | Mini-ITX-friendly PSU options |

### GbE PHY chain

```
LiteEth ── ECP5 SerDes pair ── SGMII PHY chip ── magnetics ── RJ45 ── 1000BASE-T
(fabric)   (1.25 GBaud)        (Microchip
                                KSZ9031RNX
                                or equivalent)
```

The SGMII PHY chip needs **2.5 V** (analog) and **1.0 V** (digital
core) rails, both added to the rev-A power tree below.

### Layer stackup target

8-layer stackup (DDR3L controlled-impedance routing drives the layer
count; power planes re-split to fit the SGMII PHY rails):

1. Top signal (high-speed)
2. Ground plane
3. Inner signal 1
4. Power plane (3.3 V / 2.5 V / 1.8 V split)
5. Power plane (1.35 V DDR3L+VCCIO / 1.0 V split — note: a single
   1.35 V rail now feeds both the DDR3L module VDD and the ECP5 IO
   bank VCCIO (SSTL135). The earlier draft showed a separate
   1.5 V rail; that rail is removed because rev-A is DDR3L only.
   See [`hw/ddr3l-decision.md`](hw/ddr3l-decision.md).)
6. Inner signal 2
7. Ground plane
8. Bottom signal

### Routing rules

- **DDR3L byte lane**: 50 Ω single-ended, 100 Ω differential, length
  matched ±5 mil within byte lane, ±10 mil across address group.
  IO standard `SSTL135_I` (single-ended) and `SSTL135D_I`
  (differential, DQS / CK pairs) — same as the OrangeCrab,
  Lattice Versa-ECP5, and Trellis Board reference designs.
- **GbE SGMII diff pair**: 100 Ω differential, length matched
  ±5 mil within pair (1.25 GBaud — looser tolerance than PCIe Gen3).
- **Inter-card MAST-link diff pairs**: 100 Ω differential, length
  matched ±2 mil within pair (target rate set by the inter-card
  ADR; same constraint as PCIe Gen3 as a defensive default).

### Power tree

- 12 V input (ATX 4-pin or barrel jack)
- 3.3 V step-down — FPGA I/O bank rails (non-DDR banks)
- 2.5 V step-down — SGMII PHY analog supply (KSZ9031RNX VDDA_2V5)
- 1.8 V step-down — DDR3L controller logic, FPGA aux bank
- **1.35 V step-down — DDR3L module VDD *and* ECP5 IO bank VCCIO
  (SSTL135) for the SO-DIMM byte lanes.** This single rail replaces
  the earlier draft's separate "1.5 V DDR3 main rail" entry. DDR3L
  consumes 1.35 V (`SSTL135`) for both the module VDD and the FPGA
  IO bank that drives it.
- 1.35 V (or chip-spec VCCINT) LDO — FPGA core. Final VCCINT value
  to be confirmed against the LFE5UM5G-85F-8BG756I datasheet during
  schematic capture (Stays #6); whether VCCINT shares the DDR3L
  1.35 V rail or has a dedicated supply is a schematic-capture
  decision, not a doc-only one.
- 1.0 V step-down — SGMII PHY digital core (KSZ9031RNX VDDC_1V0)
- 0.675 V LDO — DDR3L VTT termination (= VDD/2 of the 1.35 V
  module rail; was 0.75 V in the standard-DDR3 draft).

(Topology specifics — buck-converter part choices, bypass caps,
pour shapes — land in the schematic.)

### Bring-up checklist

(Filled in by the bring-up notebook in `bringup/`.)

## Future revs

- **rev B**: CertusPro-NX (or successor) + DDR4 SO-DIMM, PCIe Gen3
  host link via LitePCIe's mature `lfcpnxpciephy.py` wrapper (the
  open-toolchain CertusPro-NX path already in production use).
  Form factor returns to M.2 as a plug-in compute module on a
  Mini-ITX (or other) carrier.
- **rev C**: DDR5 SO-DIMM. Toolchain trajectory per the open-FPGA
  ecosystem commitment in
  `project_mission_and_open_fpga_commitment.md`.
