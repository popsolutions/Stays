<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# 2026-05-06 — LiteDRAM ECP5 DDR3 PHY status

**Status:** Production-mature. **Integration-only — no upstream
contribution gap identified for rev-A.** The PHY exists, has shipped
in three reference designs since 2019-2020, and has been quiescent
(no functional changes) since 2022. The cooperative integrates,
documents, and credits upstream.

## Upstream project

`enjoy-digital/litedram` — small-footprint configurable DRAM core
powered by Migen / LiteX. License: BSD-2-Clause (per the file header
of `phy/ecp5ddrphy.py`; the repo's top-level `LICENSE` is currently
NOASSERTION on the GitHub metadata side, but every PHY source file we
inspected carries an explicit BSD-2-Clause SPDX identifier).

Stars: 499. Default branch: `master`. Last push: 2026-05-04. CI
enabled (`.github/workflows/ci.yml`).

## Bug / feature gap

**None blocking rev-A.** Unlike the LitePCIe situation
(`2026-05-05-litepcie-ecp5phy.md`), `litedram/phy/ecp5ddrphy.py`
exists in upstream:

- **File:** `litedram/phy/ecp5ddrphy.py` (19,802 bytes,
  sha `5c516c66`)
- **Authors:** David Shah `<dave@ds0.me>` (also a key prjtrellis
  contributor) and Florent Kermarrec `<florent@enjoy-digital.fr>`.
- **Header docstring:** "1:2 frequency-ratio DDR3 PHY for Lattice's
  ECP5 — DDR3: 800 MT/s".
- **Last functional commit:** 2022-03-22 (`Reduce rdly to 3-bit`).
  All 2023+ commits are linting / import-path migrations, not
  functional changes. The PHY is **quiescent** in the
  positive sense: stable enough that nobody is patching it.

## Project context

- Surfaced in Agent 4's first ecosystem-health survey,
  `docs/upstream-contributions/0001-rev-a-known-upstream-issues.md`,
  in the LiteDRAM section. That survey already flagged six open
  issues against LiteDRAM as rev-A-relevant. Day-1 recon (this
  document) confirms that **none of the six are blockers**, but two
  are worth tracking during bring-up:
  - **#338** (wb_ctrl ports + native user port topology on
    OrangeCrab02-25F) — same topology rev-A is likely to use.
  - **#345** (DDR3 activates twice for first write) — DDR3
    protocol-level bug class, reported on Arty (Xilinx), but the
    DDR3-controller layer is shared, so it could surface on rev-A
    too.
- Cross-stream relevance: `MAST` PR #19 (merged earlier today on the
  RTL stream) introduced an `axi4_mem_model` mock backing store for
  Spanker. That mock will be replaced by a real LiteDRAM-driven AXI4
  subordinate in a future Stream 2 / Stream 3 integration PR. This
  recon clears the upstream half of that integration.
- ADR-001 (rev-A FPGA target) explicitly anchors rev-A on
  "DDR3 SO-DIMM (acoplável), single channel" with ECP5-85F. The
  upstream PHY supports exactly that combination.

## Day-1 recon (2026-05-06)

Performed via `gh api` reads against `enjoy-digital/litedram` and
`litex-hub/litex-boards`; no clone, no fork created (none needed —
nothing to patch).

### 1. Upstream PHY exists and is mature

- `litedram/phy/` directory listing confirms `ecp5ddrphy.py` is
  shipping alongside `s7ddrphy.py` (Xilinx 7-Series), `usddrphy.py`
  (UltraScale/UltraScale+), `gw5ddrphy.py` (Gowin GW5A), `gw2ddrphy.py`
  (Gowin GW2A), `s6ddrphy.py` (Spartan-6), and the LPDDR4 / LPDDR5
  subdirs.
- File header asserts `Copyright (c) 2019 David Shah` —
  **seven years of in-the-wild use** at the time of writing.
- Commit log shows functional development clustered in
  2019-Q4 → 2022-Q1, then dormancy. Read as: **stable**.

### 2. DDR3 SO-DIMM compatibility

The PHY accepts standard DDR3 module timing classes from
`litedram.modules`. The `MT41J128M16` / `MT41K128M16` class declares
all four standard DDR3 speedgrade tables (800 / 1066 / 1333 / 1600
MT/s), so SO-DIMM modules with any of those speedgrades will
configure cleanly at the timing-table level.

**The practical clock ceiling on ECP5 is well below 1600 MT/s.**
Production reference designs run the PHY at:

| Board | sys_clk_freq | DRAM clock | Effective MT/s |
|---|---|---|---|
| Lattice Versa-ECP5 (LFE5UM5G) | 75 MHz | 150 MHz | 300 |
| Trellis Board (LFE5UM5G-85F) | 75 MHz | 150 MHz | 300 |
| OrangeCrab (LFE5U-25F / 85F) | 48 MHz | 96 MHz | 192 |

The PHY's own header advertises "DDR3: 800 MT/s" — i.e. it has been
exercised up to ~400 MHz DRAM clock = sys_clk_freq=200 MHz with the
1:2 ratio. **No production design we found runs at that ceiling.**
This matters for ADR-001's "DDR3-1600 single channel = 12.8 GB/s
peak" claim:

- Theoretical DDR3-1600 = 12.8 GB/s on x64 = 1.6 GT/s × 8 bytes.
- Realistic ECP5+open-toolchain ceiling ≈ 800 MT/s (per PHY header)
  → 6.4 GB/s on x64 SO-DIMM, **half** the ADR-001 number.
- Production-validated ceiling ≈ 300 MT/s → 2.4 GB/s on x64.

This is an integration concern for Agent 2 (rev-A bring-up bandwidth
budget), **not a contribution gap**. The PHY is doing what its
silicon target permits. Filed for cross-stream visibility.

### 3. IO standard compatibility

ECP5's left/right IO banks support **SSTL135_I** (DDR3L 1.35V)
natively. All three production reference designs declare DDR3 pads
as `SSTL135_I` (single-ended) and `SSTL135D_I` (differential, used
for `dqs_p/n` and `clk_p/n`):

```python
# from litex_boards/platforms/lattice_versa_ecp5.py
("ddram", 0,
    Subsignal("a",     Pins("..."), IOStandard("SSTL135_I")),
    Subsignal("ba",    Pins("..."), IOStandard("SSTL135_I")),
    Subsignal("dq",    Pins("..."), IOStandard("SSTL135_I")),
    Subsignal("dqs_p", Pins("..."), IOStandard("SSTL135D_I"),
    Subsignal("clk_p", Pins("..."), IOStandard("SSTL135D_I")),
    ...
```

**Implication for Agent 2's SO-DIMM connector choice:** rev-A must
target a **DDR3L (1.35V)** SO-DIMM, not standard DDR3 (1.5V). The
Micron `MT41K*` family used by OrangeCrab is DDR3L; the `MT41J*`
family used by Trellis Board is standard DDR3 1.5V — and Trellis
Board uses `SSTL135` anyway (under-volting the IO).

ECP5's IO banks **cannot** drive 1.5V SSTL15 DDR3 directly without
an external level-shifter, so DDR3L is the correct path. This is
consistent with what the open-FPGA ECP5+DDR3 ecosystem has settled
on, but it is worth Agent 2 confirming explicitly in the rev-A BOM.

### 4. Production usage signals

Three independent reference designs use `ECP5DDRPHY` with DDR3 in
`litex-hub/litex-boards`:

| Board | Path | Module | sys_clk |
|---|---|---|---|
| Lattice Versa-ECP5 | `targets/lattice_versa_ecp5.py` | `MT41K64M16` | 75 MHz |
| Trellis Board | `targets/trellisboard.py` | `MT41J256M16` | 75 MHz |
| OrangeCrab | `targets/gsd_orangecrab.py` | `MT41K64M16/128M16/256M16/512M16` | 48 MHz |

(ULX3S and Colorlight i5/i9 also ship LiteX targets, but they use
SDR SDRAM (`MT48LC16M16`, `M12L64322A`) via `gensdrphy`, not
`ECP5DDRPHY`. They are not DDR3 references.)

OrangeCrab in particular has been the daily-driver ECP5+DDR3 board
for the LiteX community since ~2020 and ships in revisions 0.1 / 0.2
to thousands of users (Group Get / GroupGets / OSH Park retail).
That is real-world burn-in.

### 5. Known gaps / warnings (open upstream issues)

From the broader 54-open-issues list on `enjoy-digital/litedram`, the
following are tagged ECP5 and/or DDR3 and worth Agent 2/3 attention
during bring-up:

| # | Title | Recency | Severity |
|---|---|---|---|
| 194 | ECP5: Use ODDRX1F for address and command bus | 2024-06 | Enhancement (constructive — author offered a PR) |
| 338 | wb_ctrl ports of ECP5 litedram_core failing when user_ports is native | 2023-05 | Topology bug — same topology rev-A likely uses |
| 329 | ulx3s example does not work | 2023-04 | Likely user-config issue, not PHY bug (ULX3S uses SDR, not DDR3 — issue title is misleading) |
| 345 | LiteDRAM DDR3 issues activate command twice for write | 2023-08 | DDR3-protocol bug class. Reported on Arty (Xilinx), but the controller layer is shared. Potential reproducer source for us |
| 344 | LiteDRAM DDR3 AXI read data appears on Native port | 2024-04 | AXI binding bug. Affects Wishbone↔AXI interop. Reproducer worth filing if we hit it on rev-A |
| 342 | AXI port write data error | 2023-06 | AXI subsystem stability |

None of these are PHY-level blockers. The pattern is: **the PHY is
solid; the surrounding controller / AXI bridge layer has a few open
corner cases.** Agent 2 should bookmark them during bring-up, and
Agent 4 (this lane) should keep watching for new entries.

## Reproducer (minimal LiteX command)

Once Agent 2 has a rev-A platform Python file
(`stays/platforms/popsolutions_innerjib7ea.py`, conventional name
under `litex-hub/litex-boards` style) declaring the SO-DIMM signals
under `("ddram", 0, ...)` with `IOStandard("SSTL135_I")`, the PHY
instantiates with:

```python
from litedram.modules import MT41K128M16  # or whichever DDR3L SO-DIMM module
from litedram.phy import ECP5DDRPHY

# inside BaseSoC.__init__:
self.crg = _CRG(platform, sys_clk_freq=50e6)  # generates sys, sys2x, sys2x_i, init clocks
self.ddrphy = ECP5DDRPHY(
    pads         = platform.request("ddram"),
    sys_clk_freq = 50e6,  # → 100 MHz DRAM clock → 200 MT/s — conservative for rev-A bring-up
)
self.add_sdram(
    "sdram",
    phy           = self.ddrphy,
    module        = MT41K128M16(50e6, "1:2"),  # SO-DIMM module class
    l2_cache_size = 8192,
)
```

This is the same pattern as `litex_boards/targets/gsd_orangecrab.py`
lines ~80-110. **Stream 2 should mirror that target file structure
when it adds the rev-A LiteX target.**

## Status of the upstream PHY's maturity

**Production. Stable. No active defects on the PHY itself.**

Maturity signals:
- 7 years of in-the-wild use (2019 → 2026)
- 3 actively maintained reference targets in `litex-hub/litex-boards`
- Authored by David Shah (one of the prjtrellis architects) — i.e.
  the same person who reverse-engineered the ECP5 bitstream wrote
  the DDR3 PHY for it
- Last functional commit 2022-Q1; only lint/import-path commits
  since then
- Zero open issues against `phy/ecp5ddrphy.py` specifically

## Resolution

**No upstream contribution required for rev-A.** This is an
integration-only item. Stream 2 will instantiate the PHY as part of
the rev-A LiteX target file (future PR); Stream 3 (Spanker) will
then talk to the resulting AXI4 subordinate from kernel-space and
retire the `axi4_mem_model` mock that PR #19 wired up.

**What we do owe upstream, per the project mission
(`project_mission_and_open_fpga_commitment.md`):**

1. **Credit.** When rev-A bring-up succeeds on real silicon, file an
   "in-the-wild" notice via the upstream issue tracker (or
   discussion if enabled by then) confirming PopSolutions
   InnerJib7EA-rev-A boots LiteDRAM on ECP5-85F with a stock
   DDR3L SO-DIMM. This costs nothing and gives David Shah +
   Florent Kermarrec a citable consumer.
2. **Reproducers if we hit one of the known-issue corner cases.**
   Especially #345 (DDR3 first-write activate) and #344 (AXI vs
   Native port routing). If our cocotb/Verilator harness reproduces
   either, that is exactly the kind of upstream gift the mission
   memo asks for.
3. **Watch for new issues** during quarterly ecosystem-health survey
   (next: 2026-08-05). If a new defect lands that affects rev-A,
   Agent 4 picks it up under a new contribution-log entry.

The project_mission_and_open_fpga_commitment.md commitment to
"upstreaming fixes is mission-level" remains active — it just
doesn't have a target on this dependency right now. Real work
exists elsewhere (LitePCIe ECP5 PHY contribution, the open
LitePCIe scoping discussion at `enjoy-digital/litepcie#164`, plus
the rev-A bring-up issues this survey will surface in the coming
months).

## Upstream link

No upstream issue or PR filed by Agent 4 against
`enjoy-digital/litedram` as a result of this recon — there is no
gap to file against. The links worth bookmarking:

- Repo: `https://github.com/enjoy-digital/litedram`
- File: `https://github.com/enjoy-digital/litedram/blob/master/litedram/phy/ecp5ddrphy.py`
- Reference target: `https://github.com/litex-hub/litex-boards/blob/master/litex_boards/targets/gsd_orangecrab.py`
- Open-issue watchlist (DDR3/ECP5/AXI): #194, #338, #344, #345, #342

## Resolution status

- **2026-05-06:** Day-1 recon complete. PHY confirmed
  production-mature; no upstream contribution opportunity for rev-A.
  Status set to **integration-only**. Stream 2 cleared to instantiate
  the PHY in the future rev-A LiteX target PR. Quarterly ecosystem
  re-survey will keep this entry fresh.

Authored by Agent 4 (Open FPGA Upstream Contributions).
