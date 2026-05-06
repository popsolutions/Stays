<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# ADR-001 — FPGA target for the bootstrap PCB

**Status:** Accepted (2026-05-05)

## Context

The cooperative needs a custom open-hardware FPGA-based PCB that:

- Validates MAST RTL on real hardware (not just Verilator simulation)
- Has **socketed memory** (DDR SO-DIMM acoplável) so users can upgrade
  RAM, validating the project's Global-South-friendly upgradeability
  thesis
- Uses an **end-to-end open-source toolchain** so contributors anywhere
  can reproduce the build without paying for proprietary licenses
- Ships **fast enough** to address pain today, accepting trade-offs
  per the mission: a usable inferior beats an unusable superior

The decision space (filtered by "DDR + open toolchain + commercially
available"):

| FPGA | Open toolchain | DDR support | PCIe | Maturity |
|---|---|---|---|---|
| Lattice ECP5-85F | ✅ yosys + nextpnr-ecp5 + prjtrellis (rock solid) | DDR3 only | Gen2 hard PCS only (no open-toolchain TLP/DLL/LTSSM stack — see Why GbE below) | mature |
| Lattice CertusPro-NX | ⚠️ Project Oxide (still maturing) | DDR4 | Gen3 hard IP | nascent |
| Xilinx Artix-7 | ❌ Vivado | DDR3 only | Gen2 | proprietary |
| Xilinx Kintex-7 | ❌ Vivado | DDR4 | Gen3 | proprietary |

## Decision

**FPGA target: Lattice ECP5-85F (LFE5UM5G-85F-8BG756I)**.

**Memory: DDR3 SO-DIMM (acoplável), single channel.**

**Host link (rev A): GbE 1000BASE-T via LiteEth on the ECP5 SerDes pair
→ external SGMII PHY → RJ45 with integrated magnetics.** (Pivot from
the originally-locked PCIe Gen2; see "Why GbE for rev-A" below. PCIe
returns in rev B with CertusPro-NX, where LitePCIe already ships
`lfcpnxpciephy.py` upstream and the open-toolchain PCIe path is in
production use.)

**Form factor (rev A): Mini-ITX SBC, 170 mm × 170 mm.** (M.2 22110
cannot accommodate the DDR3 SO-DIMM connector + RJ45 magnetics +
inter-card edge + JTAG + USB-UART. M.2 returns in rev B as a plug-in
compute module.)

**Toolchain: yosys + nextpnr-ecp5 + prjtrellis + LiteDRAM + LiteEth
(rev A) / LitePCIe (rev B onward, via Agent 4 upstream contribution).**

**Roadmap commitment**: this is rev A, not the destination. Memory tier
moves forward in lockstep with the open FPGA ecosystem maturing:

- **Rev A (now)**: ECP5 + DDR3 SO-DIMM. Ships in 6-9 months for first
  working PCB. Validates MAST RTL on real fabric, validates the SO-DIMM
  upgradeability thesis, validates the inter-card connector idea.
- **Rev B (when Project Oxide matures)**: CertusPro-NX or successor +
  DDR4 SO-DIMM. We may help Project Oxide mature by upstreaming bugs
  and recipes we find.
- **Rev C (longer term)**: DDR5. Same trajectory.
- **Beyond**: DDR6 is already on the proprietary horizon. The open
  ecosystem must catch up, and we contribute to that.

## Consequences

### Wins

- **Open toolchain end-to-end.** Any cooperative member, anywhere, can
  reproduce the build with `apt install yosys nextpnr-ecp5 ...` plus a
  KiCad install. No licence dance, no Vivado download, no proprietary
  tool dependency.
- **DDR3 SO-DIMM is acoplável from rev A.** Validates the "user can
  upgrade RAM" thesis at the cheapest moment in the project's life.
- **Time to first working PCB drops to ~6-9 months** vs ~12-15 months
  for a CertusPro-NX-based board where Project Oxide DDR4 PHY is still
  catching up. The mission says "speed of access wins".
- **GbE host link is enough for the rev-A reference workload.**
  TinyLlama-1.1B weight load is 700 MB in ~6 s; token streaming is
  far below 1 Gbps. LiteEth on ECP5 is upstream-mature today, while
  LitePCIe has no ECP5 PHY (see Agent 4's rev-A upstream survey,
  `docs/upstream-contributions/0001-rev-a-known-upstream-issues.md`,
  "Top finding"). Choosing GbE keeps rev A on the open-toolchain
  path without blocking the MVP on multi-quarter upstream work.

### Trade-offs accepted

- **DDR3, not DDR4.** When migrating to silicon, LiteDRAM config changes
  (DDR3 PHY → DDR4 PHY); the AXI4 boundary is identical. No RTL change
  to the rest of the system.
- **Lower bandwidth.** DDR3-1600 single channel = 12.8 GB/s peak vs
  DDR4-3200 dual = 51.2 GB/s. Sufficient for inference of <1B parameter
  models in rev A; the larger Sails get more memory bandwidth in their
  silicon target without needing it in this prototype.
- **GbE not PCIe for rev A.** 1 Gbps host link instead of PCIe's
  GB/s class. Adequate for sub-1B-parameter inference (TinyLlama-1.1B
  token-streaming workloads). Larger workloads wait for rev B's PCIe
  Gen3 path. Trade-off is intentional and time-boxed to rev A.

### Operational

- KiCad project for the PCB lives under `kicad/innerjib7ea-rev-a/`.
- The first KiCad commit kicks off Sprint G in the InnerJib7EA roadmap.
- Form factor (rev A): Mini-ITX SBC. Board layout details
  (connectors, layer stackup, GbE PHY chain, power tree) live in
  `docs/PCB_DESIGN.md` rev-A subsection.
- Inter-card connector decision is a separate ADR (this one focuses on
  the FPGA chip, memory tier, host link, and form factor).
- BOM target: ~R$ 1500-2500 per assembled board, in 5-10 unit batches.

## References

- Project memory: mission and open-FPGA-ecosystem commitment
- Project memory: multi-card parallelism is first-class
- popsolutions/MAST issue #9 — ADR-014 inter-card link architecture
- popsolutions/MAST issue #13 — host link pivot rev-A PCIe → GbE
- popsolutions/InnerJib7EA issue #5 — FPGA target board (now resolved
  to: design our own per this ADR, not buy ULX3S/Arty/Kintex)
- popsolutions/InnerJib7EA issue #8 — PCB inter-card connector pinout
- popsolutions/Stays issue #15 — host link amendment tracking
- popsolutions/Stays issue #16 — form factor amendment tracking
- `docs/upstream-contributions/0001-rev-a-known-upstream-issues.md` —
  Agent 4's first ecosystem-health survey; the "Top finding (CRITICAL
  for rev A)" section enumerates the three paths (author the PHY
  upstream / switch host link off PCIe / defer PCIe to rev B) that
  the GbE pivot below resolves to.

## Why GbE for rev-A (host-link pivot rationale)

Per `project_mission_and_open_fpga_commitment.md` —
*"precarious-but-available beats premium-but-locked"*.

The original ADR-001 Context table called the ECP5's PCIe support
"Gen2 hard IP". The chip does have a hard PCS (physical-coding
sublayer; SerDes + 8b/10b + framing), but the higher PCIe layers
— LTSSM, DLL, TLP, flow-control credits — are **not** in a hard
block; they need a soft controller. The only widely-used soft
controller for the ECP5 PCIe path comes from Lattice's closed
Diamond toolchain. There is no open-source ECP5 PCIe controller
in production use today, and Agent 4's rev-A upstream survey
(`docs/upstream-contributions/0001-rev-a-known-upstream-issues.md`,
"Top finding") confirms that LitePCIe has no `ecp5pciephy.py`
module and no open issues / PRs targeting one.

Agent 4's survey enumerated three paths; this ADR amendment
chooses among them:

1. **Author `ecp5pciephy.py` upstream** — substantial work
   (multi-quarter, since the contributor would have to wrap or
   re-implement the LTSSM/DLL/TLP stack against the ECP5 PCS).
   Blocks the rev-A MVP for that duration.
2. **Switch rev-A host link off PCIe** — USB3 (FT601) or GbE
   (LiteEth). Both are upstream-mature on ECP5. Drops bring-up
   risk; bandwidth becomes the limit.
3. **Defer PCIe to rev B with CertusPro-NX** — rev B already
   targets CertusPro-NX, where LitePCIe's `lfcpnxpciephy.py` is
   shipping and supported.

This ADR amendment chooses **path 2 + path 3 in combination**:
GbE for rev A via LiteEth → external SGMII PHY → RJ45 (technically
sufficient for the rev-A reference workload — TinyLlama-1.1B:
700 MB weight load in ~6 s; token streaming ≪ 1 Gbps), and PCIe
returns in rev B on CertusPro-NX where the open-toolchain path
already exists. Path 1 (`ecp5pciephy.py` upstream) is not a
prerequisite for either rev; if Agent 4 chooses to author it, it
becomes a forward-looking gift to the open ecosystem rather than a
rev-A blocker.

## Amendments

- **2026-05-05** — Host link: PCIe Gen2 → GbE (LiteEth via ECP5
  SerDes → SGMII PHY → RJ45) for rev A. PCIe returns in rev B.
  See "Why GbE for rev-A" section above and popsolutions/Stays
  issue #15.
- **2026-05-05** — Form factor: implicit M.2 22110 → explicit
  Mini-ITX SBC (170 mm × 170 mm) for rev A. M.2 returns in rev B
  as a plug-in compute module. See popsolutions/Stays issue #16.
