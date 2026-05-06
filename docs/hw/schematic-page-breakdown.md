<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# rev-A schematic page breakdown — InnerJib7EA POPC_16A

**Status:** Day-1 planning (2026-05-06). Closes Stays issue #6
*planning artefact only* — actual schematic placement lands in
follow-up PRs `#6b`, `#6c`, … one page per PR for reviewability.

> **Scope.** This document enumerates the eight schematic pages
> that the rev-A KiCad project (`kicad/innerjib7ea-rev-a/`) will
> grow to. It is the contract that subsequent placement PRs honour.
> No actual symbol placement, no wire routing, no part-number
> commitments beyond what ADR-001, `PCB_DESIGN.md`, and
> `hw/ddr3l-decision.md` already lock.

## Why eight pages

KiCad 8 hierarchical schematics scale comfortably to ~10 pages of
≤ A3 sheet size before the eye loses the block diagram. Eight pages
maps cleanly onto the rev-A functional partitioning: one page per
major subsystem, plus a top-level block diagram. Each page becomes
its own PR so reviewers can inspect ~150–250 nets of context at a
time rather than the full ~1.5k-net board.

## Page map

| Page | Title | PR target | Approx. nets | Critical references |
|---|---|---|---|---|
| P1 | Block diagram (top sheet) | #6b | hierarchical only | this doc |
| P2 | ECP5-85F core | #6c | ~250 (BGA-756 power + ground + decoupling stubs) | FPGA-DS-02012-3.3, FPGA-TN-02038, FPGA-TN-02206 |
| P3 | DDR3L SO-DIMM | #6d | ~100 (address / data / strobe / VTT) | `hw/ddr3l-decision.md`, `PCB_DESIGN.md` |
| P4 | GbE host link | #6e | ~40 (SGMII + MDIO + RJ45 + magnetics) | `PCB_DESIGN.md` GbE chain |
| P5 | Inter-card connector | #6f | ~50 (MAST-link diff pairs + sideband) | PR #11 (Hirose FX18-40P) |
| P6 | USB-UART debug + JTAG | #6g | ~20 (USB D±, UART TXD/RXD, JTAG 2x5) | `PCB_DESIGN.md` connectors table |
| P7 | Power tree | #6h | ~80 (12 V → 7 rails + sequencing) | `PCB_DESIGN.md` power tree, `hw/decoupling-topology.md` |
| P8 | Boot flash | #6i | ~15 (SPI x4 + CS# + WP# + HOLD#) | ECP5 sysCONFIG (FPGA-TN-02018) |

Total: ~555 placed nets across the eight pages, plus power-rail
nets that span pages via the global `pwr` netclass.

---

## P1 — Block diagram (top sheet)

**Purpose.** Single-page mental map: anyone opening the project sees
the rev-A architecture in one frame, with arrows pointing into each
sub-sheet. This is the only page that contains no real components —
it is hierarchical sheet symbols only.

**Contents.**

- One sheet symbol per page P2–P8.
- Inter-sheet labels:
  - `VCC_*` rails (drawn as wide buses).
  - DDR3L byte-lane buses (`DQ[15:0]`, `DQS[1:0]`, `ADDR[14:0]`,
    `BA[2:0]`, `CTRL`).
  - SGMII pair (`SGMII_TX_p/n`, `SGMII_RX_p/n`).
  - Inter-card MAST-link pairs (`MAST_TX[3:0]_p/n`,
    `MAST_RX[3:0]_p/n`).
  - SPI flash bus (`SPI_*`).
  - JTAG (`TCK`, `TMS`, `TDI`, `TDO`).
  - UART (`DBG_TXD`, `DBG_RXD`).
- A title block citing ADR-001 and the date of the most recent
  amendment.

**Author cost.** Trivial once P2–P8 are placed; hierarchical sheet
symbols are auto-stubbed by KiCad as P2–P8 land.

---

## P2 — ECP5-85F core (BGA-756 power pins, decoupling spec)

**Purpose.** Place the FPGA symbol(s) and wire every power and
ground pin to the appropriate rail with the spec'd decoupling
network adjacent to each pin group. Signal pins (DDR3 byte lanes,
SGMII, MAST-link) terminate at hierarchical labels that the
sub-sheets pick up.

**Contents.**

- **U1 — LFE5UM5G-85F-8BG756I** (Lattice ECP5-5G, 85k LUT, 756-ball
  BGA). Single symbol or split-into-units (one unit per power
  domain) — author's choice when the page is drawn. Splitting by
  bank is the convention used by OrangeCrab and Versa-ECP5
  reference schematics; we follow that to ease cross-reference.
- **Power pin groupings** (per FPGA-DS-02012-3.3 §3.2 Table 3.2):
  - `VCC` (1.20 V) — FPGA core.
  - `VCCAUX` + `VCCAUXA` (2.5 V) — auxiliary, shared per Table
    3.2 footnote 2.
  - `VCCA` + `VCCHTX` (1.20 V) — SerDes power; can share the core
    rail or be filtered separately (decision deferred to placement
    PR per `PCB_DESIGN.md`).
  - `VCCHRX` (0.30–1.26 V) — SerDes RX bias; routing per
    FPGA-TN-02206.
  - `VCCIO0..7` — IO bank rails. Bank assignments:
    - DDR3L byte-lane banks: **1.35 V (`SSTL135`)** per
      `hw/ddr3l-decision.md`.
    - GbE SGMII bank: 2.5 V or 3.3 V per Lattice MIPI / SGMII app
      note (final decision lives on this page).
    - General-purpose IO: 3.3 V.
  - `GND` — single net, star-tied at the BGA fanout.
- **Decoupling network** per `hw/decoupling-topology.md`. P2 carries
  the topology stubs (one bulk + N bypass per pin group); exact
  capacitor counts and footprints land at placement.
- **Configuration pin strapping** (`CFG_0..2`, `MODE`, `INIT_B`,
  `DONE`, `PROGRAM_B`). SPI master mode strap for boot from flash
  on P8.

**Cross-page hierarchical labels.**

- DDR3L: `DDR_DQ[15:0]`, `DDR_DQS[1:0]_p/n`, `DDR_ADDR[14:0]`,
  `DDR_BA[2:0]`, `DDR_CTRL`, `DDR_CK_p/n`, `DDR_ODT`, `DDR_RESET#`.
- SGMII: `SGMII_TX_p/n`, `SGMII_RX_p/n`, `MDIO`, `MDC`, `PHY_INT#`.
- MAST-link: `MAST_TX[3:0]_p/n`, `MAST_RX[3:0]_p/n`, sideband.
- Boot flash: `SPI_CLK`, `SPI_CS#`, `SPI_MOSI`, `SPI_MISO`,
  `SPI_WP#`, `SPI_HOLD#`.
- JTAG / UART / debug.

**Author note.** The Lattice symbol for BG756 is the largest single
symbol on the board. Inventory is in
[`symbol-library-inventory.md`](symbol-library-inventory.md).

---

## P3 — DDR3L SO-DIMM (CT4G3S160BM, address/data/strobe groups, VTT termination)

**Purpose.** Wire the 204-pin SO-DIMM connector to the ECP5 IO
banks via length-grouped buses, with the VTT termination network
on this page for locality.

**Contents.**

- **J1 — Molex 78171 series** (or TE 1473005 — equivalent) 204-pin
  DDR3 / DDR3L SO-DIMM connector.
- Reference module: **Crucial CT4G3S160BM** (4 GB DDR3L-1600,
  single-rank, 1.35 V) per `hw/ddr3l-decision.md`. The schematic
  encodes the connector pinout, not the module — the module is a
  BOM-only entry.
- **Pin groups:**
  - **Address**: A0–A14, BA0–BA2, CKE0, CS0#, RAS#, CAS#, WE#,
    ODT0. IO standard `SSTL135_I`.
  - **Data lane 0**: DQ0–DQ7, DQS0_p/n (`SSTL135D_I`), DM0.
  - **Data lane 1**: DQ8–DQ15, DQS1_p/n (`SSTL135D_I`), DM1.
  - **Clock**: CK0_p/n (`SSTL135D_I`).
  - **Power**: 1.35 V VDD pins (J1 pins per JEDEC MO-244 SO-DIMM
    pinout) tied to the 1.35 V plane on layer 5.
  - **VTT termination**: 0.675 V (= VDD/2) tied via per-line
    series Rs to the address-group rail. VTT generator (LDO or
    dedicated VTT IC such as TPS51200) on this page.
  - **VREF**: 0.675 V reference for the SSTL135 receivers.
- **Hierarchical labels** import from P2:
  `DDR_DQ[15:0]`, `DDR_DQS[1:0]_p/n`, `DDR_ADDR[14:0]`,
  `DDR_BA[2:0]`, `DDR_CTRL`, `DDR_CK_p/n`, `DDR_ODT`, `DDR_RESET#`.
- **Length-match annotation** per `PCB_DESIGN.md` routing rules:
  ±5 mil within byte lane, ±10 mil across address. Annotation only
  on the schematic — actual matched routing happens at layout.
- **SPD I²C** (J1 SPD pins → I²C bus to ECP5 for SPD readout
  during bring-up).

**Note.** The Trellis Board "1.5 V module under-volted to 1.35 V"
hack is **not** an option — see `hw/ddr3l-decision.md`. The 1.35 V
rail feeds both the module VDD and the ECP5 VCCIO bank.

---

## P4 — GbE host link (Marvell 88E1512 SGMII PHY + RJ45 + magnetics)

**Purpose.** Wire the SGMII pair from the ECP5 SerDes through an
external SGMII PHY chip into the magnetics + RJ45 jack.

**Contents.**

- **U2 — Marvell Alaska 88E1512** (32-pin QFN, SGMII ↔ 1000BASE-T
  PHY). See "PHY chip selection" below for rationale.
- **T1 — Magnetics**, integrated into the RJ45 jack (HanRun
  HR911105A or Bel Fuse 0826-1G1T-23-F per `PCB_DESIGN.md`).
- **J2 — RJ45** with integrated magnetics + LEDs.
- **SGMII pair** from ECP5 SerDes channel B (or A — channel
  decision deferred to P2 placement) AC-coupled (0.1 µF series
  caps) into the 88E1512 SGMII pins.
- **MDIO + MDC** management bus to ECP5.
- **PHY power**: 2.5 V analog (shares `VCCAUX` rail per
  `PCB_DESIGN.md` power tree), 1.0 V digital core (dedicated rail
  off the buck on P7), 3.3 V IO (shares the FPGA general-purpose
  3.3 V).
- **Strap pins** for PHY mode (SGMII slave, auto-negotiation
  enabled, MDIO PHY address) per the 88E1512 datasheet hardware
  configuration table.
- **Reset and clock**: PHY 25 MHz reference crystal (or shared
  with FPGA reference clock — decision lives at placement);
  PHY_RESET# to ECP5 GPIO.

### PHY chip selection (decision made for this PR)

`PCB_DESIGN.md` previously named **Microchip KSZ9031RNX** in the
`### GbE PHY chain` block but called for **2.5 V** analog and
**1.0 V** digital rails. Both KSZ9031RNX and the **Marvell
88E1512** are commonly cited SGMII PHYs in the open-FPGA
community.

**This PR selects the Marvell 88E1512** as the rev-A reference.
Rationale:

| Criterion | KSZ9031RNX | 88E1512 | Outcome |
|---|---|---|---|
| LiteEth integration | tested, multiple boards | tested, multiple boards | tie |
| Open-source PHY init code | yes (`liteeth/phy/ks*`) | yes (`liteeth/phy/marvell*` style) | tie |
| Distributor stocking (LATAM importers) | uneven | broad (Mouser, Digi-Key, Arrow) | 88E1512 |
| Datasheet open access | NDA-free | NDA-free public datasheet (`88E151x_PB.pdf`) | tie |
| Package | 48-pin QFN, 7×7 mm | 32-pin QFN, 5×5 mm | 88E1512 (smaller, fewer balls) |
| Default 25 MHz crystal | yes | yes | tie |
| Hardware-strap auto-negotiation | yes | yes | tie |

The KSZ9031RNX remains an acceptable substitute for builders who
have it in stock; placement PR P4 documents the swap procedure
inline. The 1.0 V digital core rail and 2.5 V analog rail are
fungible between the two chips at this voltage class — no power
tree change needed if a builder swaps.

> **Cross-check.** `PCB_DESIGN.md` will be updated in a follow-up
> documentation PR (out of scope for this planning artefact) to
> normalise on 88E1512 as the primary reference and KSZ9031RNX as
> the documented substitute. This page-breakdown doc is the leading
> edge of that change.

---

## P5 — Inter-card connector (Hirose FX18-40P-0.8SH per PR #11)

**Purpose.** Wire the inter-card MAST-link diff pairs from the
ECP5 SerDes (other channel) plus sideband signals into the
40-pin inter-card connector. Multi-card aggregation requires this
connector be physically present on rev-A even if rev-A ships
single-card.

**Contents.**

- **J3 — Hirose FX18-40P-0.8SH** (40-pin 0.8 mm pitch board-to-board
  connector, low-profile, mezzanine-style). Reference from PR #11
  (the inter-card connector ADR / footprint PR — to land before P5
  placement; tracked separately).
- **MAST-link diff pairs** (4 lanes TX + 4 lanes RX):
  - `MAST_TX[3:0]_p/n` — ECP5 SerDes channel A (or B — decision
    on P2).
  - `MAST_RX[3:0]_p/n` — ECP5 SerDes paired with the TX channel.
  - AC coupling caps (0.1 µF) on each pair, on this page.
- **Sideband** (single-ended, slow):
  - `MAST_PRESENT#` — bus presence detect.
  - `MAST_INT#` — wake / interrupt to neighbouring card.
  - `MAST_RESET#` — synchronised reset propagation.
  - `MAST_I2C_SCL` / `MAST_I2C_SDA` — out-of-band management.
  - 4 reserved sideband pins.
- **Power forwarding**: 3.3 V_AUX rail for the receiving card's
  housekeeping MCU (1 A budget). The data card does not draw its
  main 12 V across this connector.
- **Mechanical-only edge fingers** per `PCB_DESIGN.md` connectors
  table — note this row in the table treats inter-card link as
  PCIe-style edge fingers; PR #11 may revise to the FX18-40P
  mezzanine option. The schematic page authors against whatever
  PR #11 lands.

> **Dependency.** P5 cannot land before PR #11 lands. P5's PR
> description must reference PR #11 explicitly.

---

## P6 — USB-UART debug + JTAG

**Purpose.** Bring-up debug surface: a USB-to-UART bridge for
console output and a 2x5 0.05" JTAG header for FPGA bitstream
debugging.

**Contents.**

- **U3 — FT232HL** (or **CP2102N** — see PHY-style decision matrix
  below). USB-to-UART (and FT232HL also offers MPSSE for JTAG
  bit-banging in a pinch).
- **J4 — Micro-USB B** receptacle. ESD diode array on D+/D- pair.
- **J5 — JTAG 2x5 header**, 0.05" pitch, shrouded, Tigard /
  Black Magic Probe / J-Link compatible pinout.
- **Decoupling**: per FT232HL datasheet (10 µF + 100 nF on
  3.3 V_USB rail; 100 nF on 1.8 V VCCIO).
- **Power**: USB 5 V comes in on J4; the LDO inside FT232HL
  provides 3.3 V locally. The board's 3.3 V_AUX rail is **not**
  fed from USB — keep them isolated so that an unplugged USB
  cable does not collapse the board's 3.3 V_AUX.

### Bridge IC selection (decision made for this PR)

`PCB_DESIGN.md` lists "FT232H or CP2102N bridge" as tentative
("revisit when bring-up scripts land"). This planning doc
selects the primary chip:

| Criterion | FT232HL | CP2102N | Outcome |
|---|---|---|---|
| Cost (single units) | $5–7 | $2–3 | CP2102N |
| MPSSE for JTAG bit-bang | yes (load via OpenOCD) | no | FT232HL |
| Driver maturity (Linux mainline) | yes | yes | tie |
| Package | LQFP-48 | QFN-24 | tie |
| Distributor stocking | broad | broad (Silicon Labs) | tie |

**Selection: FT232HL** as the primary bring-up bridge, because
the MPSSE feature lets a user without a J-Link or Tigard still
program the FPGA bitstream over USB via OpenOCD. CP2102N is the
documented BOM substitute for cost-sensitive builds that already
have a JTAG probe.

---

## P7 — Power tree (12 V → buck → 1.2 V / 2.5 V / 3.3 V / 1.35 V / 0.675 V / 1.0 V / 1.8 V)

**Purpose.** Convert the 12 V ATX / barrel jack input into the
seven secondary rails with the topology, sequencing, and bulk
filtering called for by the power tree in `PCB_DESIGN.md`.

**Contents.**

- **J6 — power input.** 4-pin Molex KK254 (ATX-style) per
  `PCB_DESIGN.md` connectors table; barrel jack as alternate
  population.
- **Bulk input filtering**: 470 µF aluminium polymer + 100 nF
  ceramic at the connector. TVS clamp (SMBJ15A) for 12 V transient
  protection.
- **Inrush current limit**: NTC thermistor (or soft-start
  buck-converter feature) on the 12 V rail.
- **Seven rails** per `PCB_DESIGN.md` power tree:
  1. **3.3 V** — buck step-down, ~3 A. Feeds FPGA `VCCIO` 3.3 V
     banks, USB-UART bridge, sideband logic.
  2. **2.5 V** — buck step-down, ~1 A. Feeds FPGA `VCCAUX` /
     `VCCAUXA` (shared per Table 3.2 footnote 2), 88E1512 analog
     supply.
  3. **1.8 V** — buck step-down, ~0.5 A. General-purpose 1.8 V
     auxiliary; **not** VCCAUX (which is 2.5 V) — see
     `PCB_DESIGN.md` rev-A subsection note.
  4. **1.35 V** — buck step-down, ~3 A. Feeds DDR3L SO-DIMM VDD
     **and** the ECP5 IO bank VCCIO for SSTL135 byte lanes
     (single rail per `hw/ddr3l-decision.md`).
  5. **1.20 V** — buck step-down, ~5 A. Feeds FPGA core `VCC`
     **and** SerDes `VCCA` / `VCCHTX` per `PCB_DESIGN.md` power
     tree (corrected from the earlier 1.35 V LDO entry — see
     `PCB_DESIGN.md` "1.2 V step-down" bullet).
  6. **1.0 V** — buck step-down, ~0.3 A. Feeds 88E1512 digital
     core.
  7. **0.675 V** — LDO or dedicated VTT regulator (TPS51200 or
     equivalent), ~0.5 A. DDR3L VTT termination (= VDD/2 of the
     1.35 V module rail).
- **Sequencing**: power-good chain (12 V_PG → 3.3 V_PG → 2.5 V_PG
  → 1.35 V_PG → 1.2 V_PG → 1.0 V_PG → board reset deasserted).
  Sequencing logic: discrete (3-input AND) or a power-management
  IC (TPS65086, LTC2937 — decision lives at placement).

> **Out of scope for this planning doc.** Exact regulator part
> numbers (e.g. TPS54302 vs LMR33630 for the 3.3 V buck) — the
> placement PR (P7) authors a decision matrix similar to the PHY
> matrix above and the BOM lands with that PR.

---

## P8 — Boot flash (SPI for ECP5 bitstream)

**Purpose.** Non-volatile bitstream storage for the ECP5. The
ECP5 sysCONFIG (FPGA-TN-02018) supports SPI master, SPIm x1 / x2
/ x4, and SlaveSPI / SlaveSerial / JTAG. rev-A uses SPIm x4
(quad-SPI) for fastest configuration.

**Contents.**

- **U4 — Cypress S25FL128SAGMFI001** (or equivalent) — 128 Mb
  (16 MB) quad-SPI NOR flash, 3.3 V, 8-pin SOIC. 16 MB is sized
  for the full 85F bitstream (~3 MB compressed) plus headroom for
  multi-config / golden-image schemes.
- **8 SPI lines**: CLK, CS#, IO0 (MOSI), IO1 (MISO), IO2 (WP#),
  IO3 (HOLD#) — quad-SPI uses all four. Strapping: ECP5 sysCONFIG
  configured for SPIm x4.
- **Decoupling**: 100 nF ceramic at the flash; 10 µF bulk on the
  3.3 V rail near the chip.
- **Pull-ups** on CS# (10 kΩ) and HOLD# (10 kΩ) so the flash is
  deselected during ECP5 reset.

> **Out of scope.** Multi-config golden-image SPI flash layout
> (one SPI flash with two bitstream slots, ECP5 watchdog falls
> back to slot 0 if slot 1 is bad). That feature is rev-B+
> material; rev-A is single-image.

---

## What this doc deliberately does **not** specify

- **Exact regulator part numbers.** The PHY and bridge IC
  decisions above are concrete because they are externally
  visible (BOM, board outline, software). Regulator selection is
  a decision matrix authored in P7's placement PR.
- **Exact decoupling capacitor counts.** Topology only — see
  [`decoupling-topology.md`](decoupling-topology.md). Counts
  land at placement.
- **Net naming convention details.** Convention is "ALL_CAPS with
  underscores; signed-end suffix `_p` / `_n`; active-low suffix
  `#`". The placement PRs follow this convention without further
  discussion.
- **Layer stackup decisions.** Already locked in `PCB_DESIGN.md`
  (8-layer); revisited only if a placement PR finds a constraint
  that breaks the locked stackup.
- **BOM file.** Generated from the placed schematic by `kicad-cli
  sch export bom`; not in scope for any of the page-placement PRs
  individually. A separate BOM PR consumes the union of P2–P8.

## References

- ADR-001 — [`../adr/0001-fpga-target.md`](../adr/0001-fpga-target.md)
- `PCB_DESIGN.md` — [`../PCB_DESIGN.md`](../PCB_DESIGN.md)
- DDR3L decision — [`ddr3l-decision.md`](ddr3l-decision.md)
- Symbol library inventory —
  [`symbol-library-inventory.md`](symbol-library-inventory.md)
- Decoupling topology —
  [`decoupling-topology.md`](decoupling-topology.md)
- Lattice **FPGA-DS-02012-3.3** — ECP5 / ECP5-5G Family Data Sheet.
- Lattice **FPGA-TN-02038** — ECP5 Hardware Checklist (decoupling).
- Lattice **FPGA-TN-02206** — ECP5 SerDes / PCS Usage Guide.
- Lattice **FPGA-TN-02018** — ECP5 sysCONFIG Usage Guide.
- Marvell **88E151x Public Brief** (`88E151x_PB.pdf`) — SGMII PHY.
- Microchip **KSZ9031RNX** datasheet — SGMII PHY substitute.
- FTDI **FT232HL** datasheet (DS_FT232H) — USB-UART bridge.
- Silicon Labs **CP2102N** datasheet — USB-UART bridge substitute.
- JEDEC **MO-244** — DDR3 / DDR3L SO-DIMM mechanical / pinout.

Authored by Agent 2 (FPGA Hardware).
