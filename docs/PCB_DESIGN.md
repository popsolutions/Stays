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

## InnerJib7EA rev A — Lattice ECP5-85F + DDR3 SO-DIMM + GbE host link

> Spec is locked by [ADR-001](adr/0001-fpga-target.md), including the
> 2026-05-05 amendments on host link (PCIe → GbE) and form factor
> (M.2 → Mini-ITX).

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
| DDR3 SO-DIMM (acoplável) | Molex 78171 series (204-pin) or TE 1473005 | unbuffered, 1.5 V |
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

8-layer stackup (DDR3 controlled-impedance routing drives the layer
count; power planes re-split to fit the SGMII PHY rails):

1. Top signal (high-speed)
2. Ground plane
3. Inner signal 1
4. Power plane (3.3 V / 2.5 V / 1.8 V split)
5. Power plane (1.5 V / 1.35 V / 1.0 V split)
6. Inner signal 2
7. Ground plane
8. Bottom signal

### Routing rules

- **DDR3 byte lane**: 50 Ω single-ended, 100 Ω differential, length
  matched ±5 mil within byte lane, ±10 mil across address group.
- **GbE SGMII diff pair**: 100 Ω differential, length matched
  ±5 mil within pair (1.25 GBaud — looser tolerance than PCIe Gen3).
- **Inter-card MAST-link diff pairs**: 100 Ω differential, length
  matched ±2 mil within pair (target rate set by the inter-card
  ADR; same constraint as PCIe Gen3 as a defensive default).

### Power tree

- 12 V input (ATX 4-pin or barrel jack)
- 3.3 V step-down — FPGA I/O bank rails
- 2.5 V step-down — SGMII PHY analog supply (KSZ9031RNX VDDA_2V5)
- 1.8 V step-down — DDR3 controller logic, FPGA aux bank
- 1.5 V step-down — DDR3 main rail
- 1.35 V LDO — FPGA core (VCCINT)
- 1.0 V step-down — SGMII PHY digital core (KSZ9031RNX VDDC_1V0)
- 0.75 V LDO — DDR3 VTT termination

(Topology specifics — buck-converter part choices, bypass caps,
pour shapes — land in the schematic.)

### Bring-up checklist

(Filled in by the bring-up notebook in `bringup/`.)

## Future revs

- **rev B**: CertusPro-NX (or successor with mature open-toolchain
  PCIe path) + DDR4 SO-DIMM, PCIe Gen3 host link via Agent 4's
  upstream `ecp5pciephy.py` contribution and / or CertusPro-NX's
  native PCIe path. Form factor returns to M.2 as a plug-in compute
  module on a Mini-ITX (or other) carrier.
- **rev C**: DDR5 SO-DIMM. Toolchain trajectory per the open-FPGA
  ecosystem commitment in
  `project_mission_and_open_fpga_commitment.md`.
