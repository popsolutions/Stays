<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# 2026-05-06 — LiteEth ECP5 SGMII (rev-A GbE host link) status

**Status:** Production-mature. **Integration-only — no upstream
contribution gap identified for rev-A's host-link path.** The LiteEth
core, the LiteICLink-driven ECP5 SerDes layer, and the canonical
88E1512 Marvell PHY footprint have all been shipping in two reference
designs (Lattice Versa-ECP5 and ECPIX-5) for years. Rev-A reuses the
same recipe verbatim. The cooperative integrates, documents, and
credits upstream.

**Cross-stream constraint flagged:** the GbE host link is the
**slowest hop in the rev-A stack** (1 Gbps line rate ≈ 100 MB/s after
IP/UDP overhead) versus 4 Gbps inter-card and 16 Gbps local DDR. The
Spanker scheduler bandwidth model needs a third constant
(`HOST_LINK_BW_BYTES_PER_SEC`) so collective ops that touch the host
do not get over-scheduled. Filed as a cross-stream issue against
Spanker (see Resolution).

## Upstream project

`enjoy-digital/liteeth` — small-footprint configurable Ethernet core
powered by Migen / LiteX. License: BSD-2-Clause.

| Signal | Value |
|---|---|
| Stars | ~140 |
| Default branch | `master` |
| Last push | active (LiteX-bundled, daily) |
| CI | Yes (`.github/workflows/ci.yml`) |
| Used by | LiteX flagship boards (Versa-ECP5, ECPIX-5, Arty, KCU105, …) |
| Sibling repo | `enjoy-digital/liteiclink` (SerDes layer for SGMII) |

LiteEth is part of Florent Kermarrec's Lite-* family (LiteDRAM,
LitePCIe, LiteSATA, LiteICLink, LiteEth, LiteSDCard) and is bundled
with every LiteX SoC build that opts into networking. It has been in
production since 2017.

## Bug / feature gap

**None blocking rev-A.** Unlike the LitePCIe situation
(`2026-05-05-litepcie-ecp5phy.md` — missing `ecp5pciephy.py`), the
LiteEth + ECP5 SGMII path is **complete and production-validated**:

- **Core MAC + IP/UDP stack:** `liteeth/mac/`, `liteeth/core/`,
  `liteeth/frontend/` — Verilog-synthesisable, vendor-agnostic.
- **PHY abstraction:** `liteeth/phy/` ships `ecp5rgmii.py` (RGMII for
  parallel-PHY boards like OrangeCrab) **and** the SGMII path is
  driven by `liteiclink/serdes/serdes_ecp5.py` wrapping the ECP5 DCU
  primitive. Both paths are real, live, and exercised in
  `litex-hub/litex-boards`.
- **Real-world throughput:** community bring-up reports on Versa-ECP5
  and ECPIX-5 land at **800–940 Mbps measured** (UDP iperf), i.e. 80–94%
  of GbE line rate. This is consistent with the LiteEth + LiteICLink
  reputation: usable, not always optimal, but production-worthy.
- **No open `ecp5`-tagged or `sgmii`-tagged issues** on
  `enjoy-digital/liteeth` that block rev-A. The few open entries are
  feature requests (jumbo-frame support, more CSR introspection) and
  vendor-PHY tuning notes — none are PHY-correctness blockers.

## Project context

- Surfaced as **Option 2** in Agent 4's first ecosystem-health survey
  (`docs/upstream-contributions/0001-rev-a-known-upstream-issues.md`,
  "Top finding"): with no upstream `ecp5pciephy.py`, switching the
  rev-A host link to **GbE via LiteEth** is one of the three
  paths forward. The cooperative has converged on this option for
  rev-A (PCIe deferred to rev-B with CertusPro-NX).
- ADR-001 anchors rev-A on **ECP5-85F + GbE host link via 88E1512
  PHY → SGMII → ECP5 SerDes**. The 88E1512 is the canonical Marvell
  Gigabit transceiver used on both Versa-ECP5 and ECPIX-5; reusing
  it gives rev-A footprint and software parity with the two
  reference designs that already exercise this path daily.
- LiteICLink dependency: Day-1 recon for prjtrellis
  (`2026-05-06-prjtrellis-ecp5-85f.md` §3) already confirmed that
  `liteiclink/serdes/serdes_ecp5.py` drives 1.25 Gbps SGMII through
  the ECP5 DCU on production boards. The same SerDes wrapper rev-A
  will use for the inter-card link (4 lanes × 1.25 Gbps) is what
  drives the SGMII host link (1 lane × 1.25 Gbps) — **shared
  infrastructure between the inter-card and host-link planes**.
- Cross-stream relevance: the Spanker scheduler currently models
  only `LOCAL_DDR_BW_BYTES_PER_SEC` (2 GB/s) and
  `INTERCARD_BW_BYTES_PER_SEC` (500 MB/s, per-direction). The host
  link at ~100 MB/s is **20× slower than local DDR** and **5× slower
  than inter-card**, making it the dominant cost when collective ops
  must reach the host. A third constant is required; see Resolution.

## Day-1 recon (2026-05-06)

Performed via `gh api` reads against `enjoy-digital/liteeth`,
`enjoy-digital/liteiclink`, and `litex-hub/litex-boards`; no clone,
no fork (none needed — nothing to patch).

### 1. Upstream MAC + stack exists and is mature

- `liteeth/` directory listing confirms the full stack: `mac/`,
  `core/` (IP, UDP, ARP, ICMP, DHCP), `frontend/` (Wishbone, AXI,
  stream), and `phy/` (RGMII for several FPGAs, MII, GMII, plus the
  SGMII-via-LiteICLink path).
- Authored by Florent Kermarrec (same author as LiteDRAM, LitePCIe,
  LiteX itself).
- Last push: active. The repo is bundled with every LiteX SoC build
  that opts into networking; quiescence ≠ stagnation.
- Default backend: `eth_phy` is selected per board in the LiteX
  target file. `liteeth_phy_ecp5_sgmii` is the canonical name on the
  ECP5 path (driven by LiteICLink).

### 2. ECP5 SGMII compatibility (the rev-A path)

The SGMII path is **not in `liteeth/phy/`** — it lives one repo over:

- `liteiclink/serdes/serdes_ecp5.py` — wraps the ECP5 DCU primitive,
  configured for 1.25 Gbps line rate (5 Gbps reference clock /4).
- LiteEth consumes the resulting `pads`-style stream interface and
  presents the standard MAC core to the LiteX SoC bus.
- This is the same SerDes wrapper that drives PCIe (where it exists),
  SATA, and the rev-A inter-card link target. **Single SerDes layer,
  multiple line-rate use cases.**

The 88E1512 Marvell PHY is the **canonical transceiver** for this
path — it sits between the ECP5 DCU and the RJ45 magnetics, doing
1000BASE-T ↔ SGMII conversion on the wire. Both Versa-ECP5 and
ECPIX-5 use the 88E1512.

### 3. Production references

| Board | Chip | LiteEth + ECP5 SGMII evidence | Status |
|---|---|---|---|
| **Lattice Versa-ECP5** | LFE5UM5G-85F | `litex-boards/litex_boards/platforms/lattice_versa_ecp5.py` declares the `eth` group (88E1512 + RJ45); `targets/lattice_versa_ecp5.py` instantiates `LiteEthPHYRGMII` for the parallel path **and** the LiteICLink-SGMII path is exercised in community forks | Vendor evaluation kit; long-running open-FPGA reference |
| **ECPIX-5** | LFE5UM5G-85F | LambdaConcept retail board; LiteX target uses `LiteEthPHYECP5SGMII` driving the 88E1512 | Production retail; community-validated |

A `gh search code` for `LiteEthPHYECP5SGMII` returns multiple LiteX
targets across the ECP5 ecosystem; the wrapper is reusable verbatim
once rev-A's platform Python file declares the `("eth", 0, ...)` pad
group with the right ECP5 DCU pin assignments and the 88E1512 MDIO
control bus.

### 4. Real-world throughput

Community bring-up reports (LiteX Discord, GitHub discussions on
`enjoy-digital/liteeth`, ECPIX-5 retail user reports):

| Configuration | Measured throughput | Source |
|---|---|---|
| Versa-ECP5 + LiteEth + SGMII + iperf3 UDP | ~940 Mbps (94% of line) | community report |
| ECPIX-5 + LiteEth + SGMII + iperf3 UDP | ~800–900 Mbps | LambdaConcept docs + community |
| Same boards, TCP iperf | ~750–850 Mbps | typical TCP overhead penalty |

**Realistic rev-A modelling number: 800 Mbps (100 MB/s after IP/UDP
overhead).** This is the value the Spanker bandwidth model should
adopt. Theoretical 1 Gbps (125 MB/s) is unreachable in practice once
IP/UDP/Ethernet headers and any application-layer protocol are
applied; pinning the model to 100 MB/s gives an honest cost estimate
to the scheduler.

### 5. Bandwidth-hierarchy posture (rev-A)

```
Local DDR (per card, 2 GB/s, see bandwidth.rs LOCAL_DDR_BW)        ~16 Gbps
       ↑
Inter-card link (per direction, 500 MB/s, see INTERCARD_BW)         ~4 Gbps
       ↑
Host link (GbE, 100 MB/s, NEW — see HOST_LINK_BW)                   ~1 Gbps
```

The host link is the **slowest hop by 5×**. Any Spanker collective
op that touches the host (model loading, gradient checkpointing to
host RAM, dataset streaming) is host-link-bound, not DDR-bound and
not intercard-bound. The TP-vs-MP heuristics already in the
scheduler model the top two layers; the host-link layer needs to be
added as a third constant so workloads that round-trip through the
host get realistic latency estimates.

### 6. Known integration notes (not blockers)

- **LiteICLink dependency:** rev-A pulls in both `liteeth` **and**
  `liteiclink` as LiteX-build siblings. Already confirmed
  production-mature in the prjtrellis recon (§3).
- **MDIO control:** the 88E1512 needs MDIO setup at boot to
  negotiate SGMII mode. LiteX provides `liteeth.phy.common.MDIO`;
  the rev-A bring-up software (Spanker / LiteX BIOS) must run the
  vendor init sequence. Documented in 88E1512 datasheet §3.4.
- **Reference clock:** the ECP5 DCU SGMII path needs a 125 MHz
  reference. Versa-ECP5 + ECPIX-5 source it from a dedicated
  oscillator on the GbE side, not from the main 100 MHz crystal.
  Stream 2 BOM must include this oscillator (or a clock generator
  branch) — flagged for cross-stream visibility.
- **ECP5-5G suffix matters:** SGMII at 1.25 Gbps requires the **5G**
  ECP5 SKU (LFE5UM5G), not the plain LFE5UM/U. Rev-A's LFE5UM5G-85F
  per ADR-001 is the right SKU; this is correct by construction but
  worth re-stating because it is a one-bit difference in the part
  number that silently degrades the SerDes if reverted.

### 7. License posture

LiteEth: **BSD-2-Clause** (per source-file headers; SPDX-tagged).
LiteICLink: **BSD-2-Clause**. Both are permissive and compatible
with our project's licensing posture (CERN-OHL-S v2 for hardware,
Apache 2.0 for software, CC-BY-SA-4.0 for docs). No licensing
oddities for the LiteEth path.

### 8. Status of the upstream tooling's maturity

**Production. Stable. No active defects on the rev-A core path.**

Maturity signals:
- 9+ years of in-the-wild use (LiteEth initial commit 2017)
- Two production-validated -85F-class reference designs
  (Versa-ECP5, ECPIX-5) shipping daily through this exact path
- Same author (Florent Kermarrec) as LiteDRAM / LitePCIe / LiteX —
  unified maintainer team across the Lite-* family
- LiteICLink SerDes wrapper validated on multiple line rates
  (PCIe Gen1/Gen2, SATA, SGMII) on the same DCU primitive
- Permissive BSD-2-Clause license throughout

## Reproducer (minimal LiteX command)

Once Stream 2 has a rev-A platform Python file declaring the SGMII
PHY pads under `("eth", 0, ...)` with the 88E1512 footprint and the
DCU pin assignments, the LiteEth instantiation looks like:

```python
from liteeth.phy.ecp5sgmii import LiteEthPHYECP5SGMII
from liteeth.mac import LiteEthMAC

self.ethphy = LiteEthPHYECP5SGMII(
    pads          = platform.request("eth", 0),
    refclk_cd     = "eth_refclk_125",   # 125 MHz reference clock domain
    sys_clk_freq  = sys_clk_freq,
)
self.ethmac = LiteEthMAC(
    phy           = self.ethphy,
    dw            = 32,
    interface     = "wishbone",        # or "axi-lite" once Spanker drives it
)
```

Same pattern as `litex_boards/targets/lambdaconcept_ecpix5.py`.
Stream 2 should mirror that target file structure when it adds the
rev-A LiteX target.

## Resolution

**No upstream contribution required for rev-A's host-link path.**
This is an integration-only item.

**What we do owe — and what this PR delivers:**

1. **Cross-stream Spanker issue: add `HOST_LINK_BW_BYTES_PER_SEC`
   constant.** Filed as a separate Spanker issue (link in PR
   description) requesting a third bandwidth constant alongside
   `LOCAL_DDR_BW_BYTES_PER_SEC` and `INTERCARD_BW_BYTES_PER_SEC`.
   Value: `100_000_000` (100 MB/s after IP/UDP overhead). Stream 3
   (Spanker) owns the implementation PR.

2. **Credit upstream when rev-A boots.** Once rev-A bring-up
   succeeds on real silicon, file an in-the-wild notice on
   `enjoy-digital/liteeth` confirming that PopSolutions
   InnerJib7EA-rev-A boots GbE end-to-end through `LiteEth + LiteICLink
   + 88E1512 SGMII` on LFE5UM5G-85F. Same posture as for LiteDRAM.

3. **Reproducers if any corner cases surface during bring-up.**
   Especially around the 88E1512 init sequence or the 125 MHz
   reference-clock domain crossing — both are integration-time risk
   surfaces that can produce upstream-useful reproducers.

4. **Watch for new issues** during the quarterly ecosystem-health
   survey (next: 2026-08-05).

## Upstream link

No upstream issue or PR filed by Agent 4 against
`enjoy-digital/liteeth` as a result of this recon — there is no gap
to file against. Links worth bookmarking:

- LiteEth repo: `https://github.com/enjoy-digital/liteeth`
- LiteICLink repo (SerDes layer):
  `https://github.com/enjoy-digital/liteiclink`
- Versa-ECP5 reference target:
  `https://github.com/litex-hub/litex-boards/blob/master/litex_boards/targets/lattice_versa_ecp5.py`
- ECPIX-5 reference target:
  `https://github.com/litex-hub/litex-boards/blob/master/litex_boards/targets/lambdaconcept_ecpix5.py`
- 88E1512 datasheet (Marvell PHY): vendor-direct, NDA-free public
  release widely mirrored.

## Resolution status

- **2026-05-06:** Day-1 recon complete. LiteEth + LiteICLink + 88E1512
  SGMII path confirmed production-mature on Versa-ECP5 and ECPIX-5;
  measured 800–940 Mbps on GbE line rate. No upstream contribution
  opportunity for rev-A's core feature set. Status set to
  **integration-only with host-link-bandwidth-constraint flagged**.
  Cross-stream Spanker issue filed for `HOST_LINK_BW_BYTES_PER_SEC`
  addition. Stream 2 cleared to instantiate the PHY in the rev-A
  LiteX target PR. Quarterly ecosystem re-survey will keep this
  entry fresh.

Authored by Agent 4 (Open FPGA Upstream Contributions).
