<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# 2026-05-06 — prjtrellis ECP5-85F bitstream tooling status

**Status:** Production-ready. **Integration-only — no upstream
contribution gap identified for rev-A's core feature set.** prjtrellis
covers all three ECP5-85F SKUs (LFE5U-85F, LFE5UM-85F, LFE5UM5G-85F)
in its device database, ships in production via OrangeCrab and Trellis
Board, and supports the DCU/SerDes primitives we need for the
inter-card link. **Watchlist** entries documented for SerDes bring-up
and a small set of unrelated open issues. The cooperative integrates,
documents, and credits upstream.

## Upstream project

`YosysHQ/prjtrellis` — documents the Lattice ECP5 bit-stream format
and provides the device database + bitstream-creation tools (`ecppack`,
`ecpunpack`, `ecpmulti`, `ecppll`) for the open ECP5 toolchain. The
"trellis" half of the canonical pipeline:

```
yosys (synth_ecp5) → nextpnr-ecp5 → prjtrellis (ecppack) → bitstream.bit
```

License: "Other" per GitHub metadata (per-file ISC / MIT-style headers
on the C++ source; prjtrellis-db ships under MIT). Maintainers: David
Shah (`gatecat`), Catherine "myrtle" (YosysHQ), with active
contributions from Miodrag Milanović, Aki Van Ness, Gwenhael
Goavec-Merou, and others.

Stars: 457. Default branch: `main`. Last push: 2026-05-01. Most
recent release: **1.4 (2023-05-16)** — but main has continued evolving
since then (last functional commit 2026-05-01: WASM exception-handling
cleanup; database update 2025-09-15). 51 open issues, 14 open PRs.

The companion device-database repo `YosysHQ/prjtrellis-db` was last
updated 2025-09-15.

## Bug / feature gap

**None blocking rev-A's core feature set.** Concretely:

- **Device DB completeness on -85F:** all three -85F variants are
  present and fuzzed:

  ```
  prjtrellis-db/ECP5/
    LFE5U-85F/    (idcode 0x41113043)
    LFE5UM-85F/   (idcode 0x01113043)
    LFE5UM5G-85F/ (idcode 0x81113043)  ← Trellis Board's chip
  ```

  Each device directory ships `tilegrid.json`, `iodb.json`, and
  `globals.json` — i.e. the bitstream is fully described.

- **DCU / SerDes support:** the upstream README explicitly lists
  "Transceivers (DCUs)" under "currently working". Fuzzers exist
  per-density: `fuzzers/ECP5/116-midmux-dcu/` ships
  `emux_25k.ncl`, `emux_45k.ncl`, **`emux_85k.ncl`** — so the DCU
  midmux is fuzzed for our exact target. The companion P&R support is
  in `nextpnr-ecp5` (`ecp5/arch.cc`, `ecp5/pack.cc`, etc.).

- **PLL / EHXPLLL primitives:** supported via `ecppll`.

- **DDR3 IO standards (SSTL135 / SSTL135D):** supported (see the
  litedram recon `2026-05-06-litedram-ecp5.md` for production
  evidence — three reference targets all use these IO standards
  through the prjtrellis flow).

- **BRAM (DP16KD):** supported.

- **MULT18X18D (DSP):** supported via manual instantiation. (Yosys-side
  inference has its own open issues — those are upstream of prjtrellis,
  not in it.)

In short: every prjtrellis feature rev-A's core path needs is already
shipping in production today. The handful of open issues in prjtrellis
itself (#180, #235, #237, #240, #241, #242, #246, #247, #248, #249)
are either MachXO2/MachXO3-specific (not on the ECP5 path), or
peripheral to rev-A's feature set (`ecpmulti` UM-85F idcode handling,
AES bitstream encryption, LUT6 instantiation, dual-output LVDS
single-ended I/O, digital temperature readout). None block rev-A
bring-up.

## Project context

- Surfaced in Agent 4's first ecosystem-health survey,
  `docs/upstream-contributions/0001-rev-a-known-upstream-issues.md`,
  in the prjtrellis section. That survey flagged four open issues
  (#160, #225, #242, #246) as worth tracking. Day-1 recon (this
  document) confirms none are blockers; #225 (`nescore.v` licensing)
  and #242 (ecppll non-determinism with `--highres`) are the two
  worth keeping in view, with workarounds.
- ADR-001 (rev-A FPGA target) anchors rev-A on **ECP5-85F** with
  the open toolchain (yosys + nextpnr-ecp5 + prjtrellis). Day-1 recon
  confirms the prjtrellis half of that anchor is sound.
- Inter-card link: per `project_multicard_parallelism.md`, rev-A
  ships with inter-card connectors. The first link plan is 4
  differential pairs at ~1.25 Gbps using the ECP5 DCU/SerDes blocks.
  Day-1 recon confirms prjtrellis fuzzes the DCU midmux on -85k and
  that LiteICLink's `serdes_ecp5.py` already drives PCIe / SATA /
  SGMII-class line rates through the open flow on production boards
  (Versa-ECP5, ECPIX-5). 1.25 Gbps SGMII is a directly supported
  preset.
- Cross-stream: future Stream 1 RTL work that lights up the inter-card
  connectors will exercise `serdes_ecp5.py` → `nextpnr-ecp5` →
  `prjtrellis`. This recon clears the prjtrellis half of that path.

## Day-1 recon (2026-05-06)

Performed via `gh api` reads against `YosysHQ/prjtrellis`,
`YosysHQ/prjtrellis-db`, `YosysHQ/nextpnr`, `enjoy-digital/liteiclink`,
and `litex-hub/litex-boards`. No clone, no fork created (none needed —
nothing to patch).

### 1. Repo health

| Signal | Value |
|---|---|
| Stars | 457 |
| Default branch | `main` |
| Last push | 2026-05-01 (WASM exception-handling cleanup PR #260) |
| Most recent tag | 1.4 (2023-05-16) |
| Last functional code change | 2026-05-01 |
| Last database update | 2025-09-15 (prjtrellis-db: MachXO2 IO encoding fix) |
| Open issues | 51 (mostly minor / non-rev-A) |
| Open PRs | 14 |
| Archived | No |
| CI | Yes (`.github/workflows/`) |

The fact that no formal release has been cut since 2.5 years before
this survey, while the `main` branch is alive and the last commit was
five days ago, fits the YosysHQ pattern: tagged releases are
infrequent because downstream consumers (yosys, nextpnr, distros)
track `main` directly. **Distro packagers track HEAD, not tags.**
Production designs build prjtrellis from `main`.

### 2. ECP5-85F device database — fully present

All three -85F SKUs are in the database:

| Device | idcode | Frames | Bits/frame | Packages |
|---|---|---|---|---|
| LFE5U-85F | 0x41113043 | 13294 | 1136 | csfBGA285, caBGA381, caBGA554, caBGA756 |
| LFE5UM-85F | 0x01113043 | 13294 | 1136 | csfBGA285, caBGA381, caBGA554, caBGA756 |
| LFE5UM5G-85F | 0x81113043 | 13294 | 1136 | csfBGA285, caBGA381, caBGA554, caBGA756 |

(Frame counts and bits-per-frame from `prjtrellis-db/devices.json`.)

The -85F shares the bitstream framing and tile grid with -45F at the
high level (same ECP5 architecture; differences are in fabric
density), so any per-tile bitstream feature working on -25F or -45F
also works on -85F. This is consistent with how prjtrellis was
reverse-engineered: a small set of tile types fuzzed once, replicated
across the full grid per-device.

### 3. DCU / SerDes primitive coverage (the inter-card link concern)

This was the most important question for rev-A given multi-card
parallelism is a first-class architectural requirement. The answer:
**supported, production-validated.**

- prjtrellis README lists "Transceivers (DCUs)" under
  *Current Status → currently working*.
- `prjtrellis/fuzzers/ECP5/116-midmux-dcu/` ships per-density
  fuzzed `.ncl` artefacts: `emux_25k.ncl`, `emux_45k.ncl`,
  `emux_85k.ncl`. The DCU midmux switching network is fuzzed for our
  exact 85k density.
- `nextpnr/ecp5/` source tree references `DCUA` in `arch.cc`,
  `pack.cc`, `globals.cc`, `gfx.cc`, and `docs/primitives.md` —
  i.e. the ECP5 DCU primitive is a first-class P&R target.
- `enjoy-digital/liteiclink/liteiclink/serdes/serdes_ecp5.py`
  (32 KB) is the production-grade Migen wrapper around the DCU.
  Stable since 2023-Q1 (last functional commit 2023-08-04 was a
  refactor to `LiteXModule`; the actual SerDes logic last changed
  2023-01-18). Supports preset line rates of:

  | Preset | Line rate | Use case |
  |---|---|---|
  | SGMII | **1.25 Gbps** | **Inter-card link target rate** |
  | SATA Gen1 | 1.5 Gbps | — |
  | PCIe Gen1 | 2.5 Gbps | — |
  | SATA Gen2 | 3.0 Gbps | — |
  | PCIe Gen2 / USB3 | 5.0 Gbps | (See LitePCIe recon for why we are not pursuing this on rev-A.) |

  rev-A's planned 1.25 Gbps inter-card link is **directly a supported
  preset** of `serdes_ecp5.py`. No upstream gap there.

- LiteICLink also ships SerDes test benches for two -85F-class boards:
  `bench/serdes/versa_ecp5.py` (Lattice Versa-ECP5, LFE5UM5G-85F) and
  `bench/serdes/ecpix5.py` (LambdaConcept ECPIX-5, LFE5UM5G-85F).
  Both exercise the full open flow including prjtrellis bitstream
  generation. rev-A is in the same chip class as both.

- Single open ECP5+DCU issue across the toolchain: `nextpnr#860`
  ("Failed to route from PCSCLKDIV primitive on ecp5", 2021,
  4 comments, last activity 2023-12). Affects designs that use
  `PCSCLKDIV` for clock-divider output from the SerDes. Workaround:
  drive the divided clock from a different primitive (e.g. fabric
  `DIV` or PLL). **Not a blocker** but worth bookmarking for the
  Stream 1 RTL author who wires the inter-card SerDes — if our design
  uses `PCSCLKDIV`, we should plan around this issue.

  (This is a `nextpnr-ecp5` routing issue, not a prjtrellis
  bitstream-format issue. Logged here because the toolchain context
  is the same and Agent 1 will hit it via the same flow.)

### 4. Toolchain version compatibility

Latest stable releases of the three pieces of the open ECP5 flow:

| Tool | Latest tag | Date |
|---|---|---|
| yosys | v0.64 | 2026-04-09 |
| nextpnr | nextpnr-0.10 | 2026-03-12 |
| prjtrellis | 1.4 | 2023-05-16 (but `main` is current; most users build from HEAD) |
| prjtrellis-db | (no tags; tracks `main`) | last update 2025-09-15 |

YosysHQ ships these together via the
[OSS CAD Suite](https://github.com/YosysHQ/oss-cad-suite-build) nightly
bundle, which is the de-facto distribution for the open FPGA flow.
The bundle pins compatible HEADs of all three. **Recommended for
rev-A bring-up:** track OSS CAD Suite nightlies (or pin to a specific
nightly) rather than mixing release tags from each project. This is
how OrangeCrab, Trellis Board, and ECPIX-5 users ship today.

### 5. Production usage signals

End-to-end production designs running through prjtrellis on -85F-class
ECP5 silicon:

| Design | Chip | Use case | Status |
|---|---|---|---|
| **OrangeCrab** | LFE5U-85F-8MG285C | Hobbyist / dev / community | **Daily-driver since 2020.** Production BOM at `orangecrab-fpga/orangecrab-hardware/hardware/orangecrab_r0.2.1/Production/`. Ships in revisions 0.1, 0.2, 0.2.1 to thousands of users via Group Get / GroupGets retail. 527 stars on the hardware repo. |
| **Trellis Board** | LFE5UM5G-85F-8BG756C | Reference/dev (gatecat's own board) | Production via @gatecat. 116 stars on `gatecat/TrellisBoard`. Trellis Board is the de-facto -85F reference platform — it is the board David Shah designed to validate his own toolchain. **The author of prjtrellis dogfoods on the same chip-class rev-A targets.** |
| **Lattice Versa-ECP5 (5G)** | LFE5UM5G-85F-8BG381 | Vendor evaluation kit | Long-running open-toolchain test target. LiteX, Migen, nMigen/Amaranth, and LiteICLink all ship reference designs against this board. |
| **ECPIX-5** | LFE5UM5G-85F-8BG554 | LambdaConcept commercial dev board | Production retail. Used as a SerDes test target in `liteiclink/bench/serdes/ecpix5.py`. |

**The cross-cutting signal:** every -85F-class ECP5 dev board sold
to the open-FPGA community for the past 5+ years runs through
prjtrellis end-to-end. This is not a fragile or experimental
dependency — it is the bedrock of the open ECP5 ecosystem.

A note on tape-out / ASIC equivalents: since the ECP5-85F is itself
a commercial Lattice silicon (not something prjtrellis has to retarget
per design), the relevant production signal is "boards that ship
ECP5-85F" rather than "designs that tape out". The four boards above
are sufficient evidence.

### 6. Known gaps / warnings (open upstream issues)

From the 51 open prjtrellis issues, filtered to those that could
intersect rev-A:

| # | Title | Repo | Status / impact for rev-A |
|---|---|---|---|
| 242 | ecppll random (uninitialised?) values in output | prjtrellis | **Workaround known.** Drop `--highres` flag — output becomes deterministic. rev-A clock budget should not require `--highres` mode (rev-A clocks are conservative). Last updated 2025-05-04. |
| 246 | LFE5U-45F-6TQ144 EHXPLLL_UR does not work | prjtrellis | Different SKU (45F-TQ144) and different PLL position (UR). **Watchlist:** keeps the question of "is the UR PLL fully exercised on -85F" open. Workaround: avoid UR PLL position (use UL/LL/LR). Last updated 2025-05-09. |
| 225 | Licensing of timing/resource/nescore.v | prjtrellis | **Meta concern.** Timing-database licensing is not crystal-clear; relevant given our CERN-OHL-S v2 hardware licensing. Not a code blocker. Last updated 2023-06. |
| 160 | PLL --highres results in multiply driven net error | prjtrellis | Same workaround as #242: drop `--highres`. 2021. |
| 860 | Failed to route from PCSCLKDIV primitive on ecp5 | nextpnr | **DCU-adjacent.** If our inter-card SerDes uses `PCSCLKDIV` for divided clock output, we hit this. Workaround: divide via fabric/PLL instead. 2021, 4 comments, last activity 2023-12. |

None of these are PHY-database-level blockers. The pattern is:
**the bitstream format and the device database are solid; a few
peripheral tools (`ecppll`, `ecppack`, etc.) and a handful of P&R
edges have known issues with simple workarounds.**

Open issues that are explicitly *not* on the rev-A path:

- #138 (Missing pins on LFE5UM5G-45F CABGA381): different package +
  different density.
- #185 (Apple M1 segfault in pytrellis): macOS-specific build issue;
  CI build host is Linux.
- #235 (MachXO2 PLL pin wires): different family entirely.
- #237 (LVDS in / single-ended out IO): unusual primitive combination
  not used by rev-A's SO-DIMM or SerDes IO.
- #248, #249, #250: minor MachXO3D and `ecppll` lint issues.
- #241 (AES bitstream encryption): out of scope per ADR-001 (we want
  open, inspectable bitstreams; encrypted bitstream is anti-mission).
- #180 (`ecpmulti` for um-85k): multi-boot bootloader edge case;
  rev-A boots from a single bitstream image at bring-up time.

### 7. License posture

prjtrellis itself: ISC-style permissive (per source-file headers),
but GitHub metadata reports "Other" because the top-level `LICENSE`
file is not a SPDX-recognised name. prjtrellis-db: MIT. Both are
compatible with our project's licensing posture (CERN-OHL-S v2 for
hardware, Apache 2.0 for software, CC-BY-SA-4.0 for docs) — we
consume them as build-time tooling, not as IP.

The only licensing oddity is the open `nescore.v` timing description
(issue #225) — a 2023 question about which license applies. Worth
revisiting if Agent 4 ever takes a serious dependency on the timing
database internals (currently we don't).

## Reproducer (minimal LiteX command)

Once Agent 1 / Agent 2 has a rev-A platform Python file declaring the
ECP5-85F target, the prjtrellis flow is invoked transparently by
LiteX's `lattice/trellis.py` build backend. Concretely:

```python
# stays/platforms/popsolutions_innerjib7ea.py (future Stream 2 work)
from litex.build.lattice import LatticePlatform
from litex.build.lattice.programmer import OpenOCDJTAGProgrammer

class Platform(LatticePlatform):
    default_clk_name   = "clk"
    default_clk_period = 1e9 / 100e6  # tentative

    def __init__(self, toolchain="trellis", **kwargs):
        LatticePlatform.__init__(
            self,
            "LFE5U-85F-8CABGA381",  # our exact part (TBD: package)
            _io,
            _connectors,
            toolchain=toolchain,    # "trellis" → yosys + nextpnr + prjtrellis
            **kwargs,
        )
```

LiteX-side examples to mirror: `lattice_versa_ecp5.py`,
`gsd_orangecrab.py`, `trellisboard.py` — all in
`litex-hub/litex-boards/litex_boards/platforms/`. **Stream 2 should
mirror that platform-file structure when it adds the rev-A platform.**

The bitstream file produced by `ecppack` is `<top>.bit`, programmed
via OpenOCD over JTAG. No further integration work needed at the
toolchain level — Stream 1's RTL and Stream 2's platform file are the
two remaining inputs.

## Status of the upstream tooling's maturity

**Production. Stable. Bedrock of the open ECP5 ecosystem.**

Maturity signals:
- 8 years of in-the-wild use (initial commit 2018-03-04, daily-driver
  for OrangeCrab + Trellis Board since 2019-2020)
- 4 actively shipping boards in the same chip class as rev-A
- Authored by David Shah (`gatecat`), the same person who maintains
  nextpnr-ecp5; full pipeline is held by the same maintainer team
- All three -85F SKUs in the device database with full bitstream
  framing data
- DCU/SerDes primitives fuzzed for -85k explicitly
- `main` branch has continuous activity (last commit 2026-05-01)
- Zero open issues against the ECP5 bitstream format itself
- Zero open issues mentioning "85F" (a clean-room search returns 0
  hits — the only "85" hit is on UM-85F idcode handling in `ecpmulti`,
  not in bitstream generation)
- Latest yosys (v0.64), nextpnr (0.10), and prjtrellis HEAD are
  designed-to-coexist via OSS CAD Suite nightlies

## Resolution

**No upstream contribution required for rev-A's core feature set.**
This is an integration-only item. Stream 1 (RTL) and Stream 2 (FPGA
HW / platform file) will exercise the prjtrellis flow as a
build-system dependency, no patches needed.

**What we do owe upstream, per the project mission
(`project_mission_and_open_fpga_commitment.md`):**

1. **Credit + in-the-wild signal.** When rev-A bring-up succeeds on
   real silicon, file a brief issue (or comment on a pinned
   discussion if the project enables Discussions) confirming the
   PopSolutions InnerJib7EA-rev-A boots end-to-end through yosys +
   nextpnr-ecp5 + prjtrellis on LFE5U-85F. This is the kind of
   real-world consumer signal the YosysHQ team explicitly values
   (David Shah's TrellisBoard repo README cites in-the-wild use).
2. **Reproducers if we hit one of the watchlist issues.** Specifically
   #242/#160 (`ecppll --highres` non-determinism), #246
   (EHXPLLL_UR / position-dependent PLL bug), or #860 (PCSCLKDIV
   routing). If our cocotb/Verilator harness or our hardware bring-up
   reproduces any of these on -85F, that is exactly the kind of
   upstream gift the mission memo asks for.
3. **Watch for new issues** during the quarterly ecosystem-health
   survey (next: 2026-08-05). If a new -85F-relevant defect lands
   that affects rev-A, Agent 4 picks it up under a new
   contribution-log entry.
4. **(Optional, post-rev-A)** When the inter-card SerDes link goes
   live: file an in-the-wild SerDes-on-prjtrellis confirmation against
   `enjoy-digital/liteiclink` and/or `YosysHQ/prjtrellis`. Production
   SerDes use cases on -85F are rarer than DDR3 use cases, so the
   signal value is higher.

The `project_mission_and_open_fpga_commitment.md` commitment to
"upstreaming fixes is mission-level" remains active — it just
doesn't have a target on this dependency right now. The real work
remains elsewhere (LitePCIe ECP5 PHY contribution at
`enjoy-digital/litepcie#164`, plus rev-A bring-up issues this and
future surveys will surface).

## Upstream link

No upstream issue or PR filed by Agent 4 against `YosysHQ/prjtrellis`,
`YosysHQ/prjtrellis-db`, or `YosysHQ/nextpnr` as a result of this
recon — there is no gap to file against. Links worth bookmarking:

- Repo (tools): `https://github.com/YosysHQ/prjtrellis`
- Repo (database): `https://github.com/YosysHQ/prjtrellis-db`
- Repo (P&R): `https://github.com/YosysHQ/nextpnr`
- Trellis Board (gatecat's reference): `https://github.com/gatecat/TrellisBoard`
- OrangeCrab hardware: `https://github.com/orangecrab-fpga/orangecrab-hardware`
- LiteICLink ECP5 SerDes: `https://github.com/enjoy-digital/liteiclink/blob/master/liteiclink/serdes/serdes_ecp5.py`
- OSS CAD Suite (recommended distribution): `https://github.com/YosysHQ/oss-cad-suite-build`
- Watchlist (rev-A relevant):
  - `prjtrellis#242` (ecppll non-determinism with --highres)
  - `prjtrellis#246` (EHXPLLL_UR positional bug)
  - `prjtrellis#225` (nescore.v licensing meta-question)
  - `nextpnr#860` (PCSCLKDIV routing on ECP5 DCU output)

## Resolution status

- **2026-05-06:** Day-1 recon complete. Tooling confirmed
  production-mature for ECP5-85F across all three SKU variants
  (LFE5U-85F, LFE5UM-85F, LFE5UM5G-85F). DCU/SerDes primitives
  supported and production-validated via LiteICLink on Versa-ECP5
  and ECPIX-5. No upstream contribution opportunity for rev-A's core
  feature set. Status set to **integration-only with watchlist**.
  Stream 1 and Stream 2 cleared to build through this dependency.
  Quarterly ecosystem re-survey (next: 2026-08-05) will keep this
  entry fresh.

Authored by Agent 4 (Open FPGA Upstream Contributions).
