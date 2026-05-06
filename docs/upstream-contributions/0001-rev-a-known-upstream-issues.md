<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# Rev-A known upstream issues survey (2026-05-05)

This is the first ecosystem-health survey conducted by Agent 4 against
the rev-A toolchain locked by ADR-001. It catalogues open upstream
issues that intersect with our configuration (ECP5-85F, DDR3-1600
SO-DIMM single channel, PCIe Gen2 hard IP) so that:

- Agents 1, 2, 3 can cite a known issue when they hit symptoms during
  bring-up, instead of re-discovering it.
- Agent 4 has a queue of upstream issues to engage with as we generate
  reproducers from our own designs.
- The cooperative has a baseline against which to detect new
  regressions in future surveys.

This survey is not authoritative — it reflects the open-issue trackers
on the date stamped above. Re-run the survey quarterly or before each
PCB rev kickoff.

## Top finding (CRITICAL for rev A)

**LitePCIe has no ECP5 PHY module.** The
`enjoy-digital/litepcie/litepcie/phy/` directory contains PHYs for
Cyclone V (`c5pciephy.py`), Gowin GW5A (`gw5apciephy.py`), Lattice
CertusPro-NX (`lfcpnxpciephy.py`), and three Xilinx variants — but no
`ecp5pciephy.py`. There are zero open issues, zero closed issues, and
zero PRs in the LitePCIe repo mentioning "lattice" or "ecp5" as of
this survey.

ADR-001 lists LitePCIe in the rev-A toolchain. With the current
upstream state, **PCIe Gen2 via the ECP5 hard IP block is not a
supported path through the open toolchain.** This is a rev-A
architectural risk that warrants a cross-stream issue (filed
separately) and likely an ADR amendment.

Three options exist; none is free:

1. **Author `ecp5pciephy.py` upstream.** Substantial work — the
   CertusPro-NX PHY is ~hundreds of LoC and wraps a Lattice IP
   instantiation. Would need to wrap the ECP5 PCS hard IP. Unblocks
   rev A and contributes a first-class capability upstream.
2. **Switch rev-A host link off PCIe.** USB3 via FT601 or gigabit
   Ethernet via LiteEth are both well-supported on ECP5. Drops bring-up
   risk but limits host-link bandwidth (USB3 ≈ 400 MB/s sustained,
   GbE ≈ 110 MB/s) versus PCIe Gen2 x1 (≈ 500 MB/s) or x4 (≈ 2 GB/s).
3. **Defer PCIe to rev B with CertusPro-NX.** Rev B already targets
   CertusPro-NX per ADR-001; LitePCIe's `lfcpnxpciephy.py` is recent
   (2024-2025) and supported. Cost: rev A loses PCIe entirely.

This decision is above Agent 4's authority (touches MVP envelope).
Filed for human + Agent R review.

## Survey by upstream project

For each project, issues are filtered to those that plausibly affect
rev-A. Issues in unrelated SKUs, packages, or configurations are
excluded. Recency is noted because old open issues with no maintainer
engagement are weaker signal than recent ones.

### yosys (https://github.com/YosysHQ/yosys)

Used for: RTL synthesis (`synth_ecp5`).

| # | Title | Affects rev-A because | Recency |
|---|---|---|---|
| 5814 | ECP5 memory_bram has no rule for REGMODE=OUTREG | MAST scratchpads / cache lines using BRAM with output register will hit this | 2026-04 |
| 4798 | Synthesis with -nowidelut gives drastically better results | QoR knob worth knowing if MAST hits area pressure on -85F | 2024-12 |
| 4872 | Yosys emits FF that never toggles instead of constant 0 | Synthesis-correctness regression class | 2025-01 |
| 4349 | Assert failure in synth_{ice40,ecp5} on simple design | Generic synth_ecp5 crash; affects any design that triggers it | 2024-04 |
| 4237 | ABC9/AIGER crash in synth_ecp5 | Crash in the default ECP5 synthesis flow | 2024-02 |
| 4127 | DSP_A/B_MINWIDTH change causes ABC9 error | Affects any design inferring ECP5 DSP blocks (MAST matrix engine candidate) | 2024-01 |
| 3008 | ECP5 primitive instantiation 'cells_not_processed' | Hits when manually instantiating ECP5 primitives | 2021-09 |
| 3005 | Lattice ECP5: Module FD1P3DX port q error | Same root area; manually instantiated FFs | 2021-09 |

### nextpnr-ecp5 (https://github.com/YosysHQ/nextpnr)

Used for: place & route on ECP5 fabric.

| # | Title | Affects rev-A because | Recency |
|---|---|---|---|
| 1642 | Adjust DELAYG/DELAYF delay value based on device size (25F/45F/85F) | DELAYG/DELAYF directly affect DDR3 byte-lane timing; 85F is our target | 2026-02 |
| 1599 | Failed to find a route for arc 0 of net | Routing-failure regression class | 2025-11 |
| 1521 | ECP5 IDDR71B not working as expected | DDR3 byte lanes use IDDR primitives; this is on the critical path | 2025-07 |
| 1469 | Differential clock pins on ECP5 EVN are not available | Diff clock input may be needed for DDR3 reference / inter-card | 2025-03 |
| 1165 | Doesn't appear to allow more than 8 global clocks | Could constrain MAST clock-tree planning | 2023-05 |
| 482 | Support BANK in ECP5 LPF parser | We will need bank constraints for the SO-DIMM IO group | 2020-07 |
| 481 | Support DEFINE PORT GROUP and IOBUF GROUP in ECP5 LPF | Constraint expressivity for grouped IOs | 2020-07 |
| 371 | ECP5 issues to pack FF in MGIOL (`syn_useioff`) | DDR3 IO timing closure depends on FF-in-IO packing | 2019-12 |

### prjtrellis (https://github.com/YosysHQ/prjtrellis)

Used for: ECP5 bitstream database + packer.

| # | Title | Affects rev-A because | Recency |
|---|---|---|---|
| 242 | ecppll random (uninitialised?) values in output | We use ecppll for clock synthesis; non-deterministic output is a CI risk | 2024-11 |
| 246 | LFE5U-45F-6TQ144 EHXPLLL_UR does not work | Different SKU, but raises the question of whether 85F EHXPLLL_UR is exercised | 2025-04 |
| 160 | PLL --highres results in multiply driven net error | High-resolution PLL output mode | 2021-01 |
| 225 | Licensing of timing/resource/nescore.v | Meta but worth tracking given our CERN-OHL-S licensing posture | 2023-06 |

### LiteDRAM (https://github.com/enjoy-digital/litedram)

Used for: DDR3 controller for the SO-DIMM.

**Day-1 recon (2026-05-06):** see
`2026-05-06-litedram-ecp5.md`. The ECP5 DDR3 PHY
(`litedram/phy/ecp5ddrphy.py`) is **production-mature**: authored by
David Shah + Florent Kermarrec in 2019, last functional commit
2022-Q1, used in three reference designs (Lattice Versa-ECP5,
Trellis Board, OrangeCrab). No upstream contribution gap for rev-A;
this is an integration-only dependency. Note: ULX3S in the table
below uses **SDR SDRAM** via `gensdrphy`, not DDR3, so issue #329 is
not on the ECP5DDRPHY path — the title is misleading.

| # | Title | Affects rev-A because | Recency |
|---|---|---|---|
| 194 | ECP5: Use ODDRX1F for address and command bus | Direct rev-A: address/command bus implementation choice for DDR3 on ECP5 | 2020-05 |
| 338 | wb_ctrl ports of ECP5 litedram_core failing when user_ports is native | ECP5 + Wishbone ctrl + native user port — the topology we're likely to use | 2023-05 |
| 345 | LiteDRAM DDR3 issues activate command twice for write | DDR3 protocol bug class | 2023-08 |
| 344 | LiteDRAM DDR3 AXI read data appears on Native port | AXI binding bug; affects Wishbone↔AXI interop choices | 2023-08 |
| 342 | AXI port write data error | AXI subsystem stability | 2023-06 |
| 329 | ulx3s example does not work | ULX3S uses SDR (gensdrphy), NOT DDR3 — title misleading; not on the ECP5DDRPHY path | 2023-04 |

### LitePCIe (https://github.com/enjoy-digital/litepcie)

Used for: PCIe Gen2 host link via ECP5 hard IP.

See top finding above. Beyond the missing-PHY structural gap, no
ECP5-relevant open issues exist. The Xilinx-centric issues
(`#162`, `#150`, `#149`, `#147`, etc.) do not apply to rev A given
the missing-PHY blocker.

## What is NOT in this survey (deliberately)

- Closed issues — too noisy without targeted reproducer work; revisit
  during reproducer development.
- Issues in repos *not* in the rev-A critical path (Verilator, cocotb,
  LiteX core, LiteEth) — those become relevant only if rev-A scope
  shifts (e.g., if option 2 above is chosen and we adopt LiteEth).
- Issues in proprietary tooling (Lattice Diamond, Vivado) — out of
  scope per ADR-001.
- The CertusPro-NX / Project Oxide ecosystem health — that's a rev-B
  survey deliverable.

## Next survey trigger

- Quarterly cadence (next: 2026-08-05), or
- Before any PCB rev kickoff, or
- Any time an Agent 1/2/3 hits an upstream gap they want triaged.

Authored by Agent 4 (Open FPGA Upstream Contributions).
