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
| Lattice ECP5-85F | ✅ yosys + nextpnr-ecp5 + prjtrellis (rock solid) | DDR3 only | Gen2 hard IP | mature |
| Lattice CertusPro-NX | ⚠️ Project Oxide (still maturing) | DDR4 | Gen3 hard IP | nascent |
| Xilinx Artix-7 | ❌ Vivado | DDR3 only | Gen2 | proprietary |
| Xilinx Kintex-7 | ❌ Vivado | DDR4 | Gen3 | proprietary |

## Decision

**FPGA target: Lattice ECP5-85F (LFE5UM5G-85F-8BG756I)**.

**Memory: DDR3 SO-DIMM (acoplável), single channel.**

**Toolchain: yosys + nextpnr-ecp5 + prjtrellis + LiteDRAM + LitePCIe.**

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
- **PCIe Gen2 hard IP is enough** for the bootstrap. The Compute Unit's
  bandwidth need is satisfied by 5 GB/s peak; we're not building a 70B
  inference rig on rev A.

### Trade-offs accepted

- **DDR3, not DDR4.** When migrating to silicon, LiteDRAM config changes
  (DDR3 PHY → DDR4 PHY); the AXI4 boundary is identical. No RTL change
  to the rest of the system.
- **Lower bandwidth.** DDR3-1600 single channel = 12.8 GB/s peak vs
  DDR4-3200 dual = 51.2 GB/s. Sufficient for inference of <1B parameter
  models in rev A; the larger Sails get more memory bandwidth in their
  silicon target without needing it in this prototype.
- **PCIe Gen2 not Gen4.** 5 GB/s host link instead of 32 GB/s. Limits
  some inference workloads but unblocks all the architectural validation.

### Operational

- KiCad project for the PCB lives under `kicad/innerjib7ea-rev-a/`.
- The first KiCad commit kicks off Sprint G in the InnerJib7EA roadmap.
- Inter-card connector decision is a separate ADR (this one focuses on
  the FPGA chip and memory tier).
- BOM target: ~R$ 1500-2500 per assembled board, in 5-10 unit batches.

## References

- Project memory: mission and open-FPGA-ecosystem commitment
- Project memory: multi-card parallelism is first-class
- popsolutions/MAST issue #9 — ADR-014 inter-card link architecture
- popsolutions/InnerJib7EA issue #5 — FPGA target board (now resolved
  to: design our own per this ADR, not buy ULX3S/Arty/Kintex)
- popsolutions/InnerJib7EA issue #8 — PCB inter-card connector pinout
