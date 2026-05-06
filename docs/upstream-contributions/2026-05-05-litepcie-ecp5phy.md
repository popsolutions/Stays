<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# 2026-05-05 — LitePCIe ECP5 PHY contribution

**Status:** Awaiting upstream feedback (outreach email sent
2026-05-06; public discussion filed — see link below).

## Upstream project

`enjoy-digital/litepcie` — small-footprint configurable PCIe core
powered by Migen / LiteX. License: BSD-2-Clause.

## Bug / feature gap

LitePCIe ships PHY modules for Cyclone V, Gowin GW5A, Lattice
CertusPro-NX, Xilinx 7-Series, UltraScale, and UltraScale+. There is
no `ecp5pciephy.py`. Zero open issues, zero closed issues, zero PRs in
the upstream repo mention "lattice" or "ecp5". The upstream README
explicitly lists "add Lattice support" under "Possible improvements",
so the contribution is welcome in principle.

## Project context

- Surfaced by Agent 4's first ecosystem-health survey:
  `docs/upstream-contributions/0001-rev-a-known-upstream-issues.md`
  (popsolutions/Stays PR #3).
- Cross-stream issue: popsolutions/MAST#13 — "[cross-stream] LitePCIe
  has no ECP5 PHY — rev-A host-link path is unsupported upstream".
- Decision (2026-05-05, hybrid path): Agent 4 pursues Option 1
  (author the PHY upstream) in parallel; Agent 2 switches the rev-A
  PCB host link to GbE via LiteEth and amends ADR-001 in
  `docs/adr/0001-fpga-target.md`. Agent 4 is no longer blocking on the
  rev-A schedule.

## Local workaround

Rev-A PCB switches host link to GbE via LiteEth (Agent 2's track).
Once Agent 2's ADR-001 amendment lands, this contribution becomes a
forward-looking upstream gift rather than a rev-A blocker — the
cooperative still ships it because authoring the missing PHY is a
mission-level deliverable per
`project_mission_and_open_fpga_commitment.md`, not a tactical
necessity.

## Day-1 recon (2026-05-05)

Performed the following before writing any code:

1. **Forked the upstream repo:**
   `https://github.com/marcos-mendez/litepcie` (no clone yet — recon
   via `gh api` reads).
2. **Read upstream README, full `phy/lfcpnxpciephy.py`** (309 lines),
   **and `phy/common.py`** (177 lines) to learn the PHY interface
   contract every module must implement.
3. **Searched upstream issues + PRs** for any prior ECP5 work — none
   exists.
4. **Searched GitHub broadly** for prior open-source ECP5 PCIe
   implementations — no clearly mature project found (LimeSDR
   variants use ECP5+PCIe via Lattice's proprietary Diamond IP;
   no open soft-stack precedent).

## Critical scope correction (the part that needs human attention)

When MAST#13 was filed, the cost estimate ("1-2 sprint-equivalents")
assumed `ecp5pciephy.py` would follow the same wrapping pattern as
`lfcpnxpciephy.py`. Day-1 recon shows that assumption is wrong.

### What the CertusPro-NX PHY actually does

`lfcpnxpciephy.py` is a Python wrapper that:

1. Downloads a **vendor-generated Verilog IP blob** as a zip from a
   GitHub user-attachment URL (`do_finalize` calls `wget` + `unzip`
   at build time).
2. Instantiates two Verilog modules from that blob (`lfcpnxpciephy`
   and `LMMI_app`) with hundreds of port mappings.
3. Wires the Lattice Memory Mapped Interface (LMMI) for runtime
   configuration.

The actual PCIe controller (LTSSM, TLP layer, Data Link Layer, MAC) is
**inside the proprietary Verilog blob**. The Python file is a
thin wrapper, hence its tractable size.

### Why ECP5 is fundamentally different

The ECP5 family has hard SerDes blocks (Dual Channel Units, "DCU")
capable of 5 Gbps PCIe Gen2 line rate, but it does **not** have an
integrated PCIe controller block. The DCU exposes the PCIe physical
layer (8b/10b coding, scrambling, framing) only — the PIPE-to-TLP
stack (LTSSM negotiation, link training, DLLP, TLP framing,
flow-control credits) must be soft-implemented in fabric.

ADR-001's phrase "Gen2 hard IP" was technically imprecise: ECP5 has a
hard PCS, not a full hard PCIe IP. Lattice's closed-source Diamond
toolchain ships a soft PCIe controller IP (in encrypted Verilog) that
sits on top of the DCU; that is what closed-source ECP5+PCIe designs
use. There is no open-source equivalent in production use today.

### Three sub-options (none costed at 1-2 sprints)

**Sub-option 1a — Wrap Lattice Diamond's closed PCIe IP**

Mirror the CertusPro-NX pattern: download a Lattice-generated soft IP
blob and wrap it in `ecp5pciephy.py`. Closed-source IP, requires
Diamond licence to regenerate, contradicts ADR-001's open-toolchain
commitment. **Rejected on mission grounds.**

**Sub-option 1b — Write a soft PCIe stack atop the ECP5 DCU**

Implement LTSSM, DLL, TLP, and the PHY adapter in Migen / LiteX, atop
the ECP5 DCU primitives. Realistic scope: months to years for a
production-grade implementation; PCIe specification is hundreds of
pages and the LTSSM alone has dozens of states with non-trivial timing
requirements. This is not 1-2 sprints; it is a major project on its
own.

**Sub-option 1c — Port an existing open-source soft PCIe stack**

Find a permissively licensed open soft PCIe stack and adapt it to
target the ECP5 DCU. Candidate projects exist in academic / research
contexts (e.g., NoC-PCIe variants, Pulpino-PCIe sketches), but none
are known production-mature. Realistic scope: weeks of evaluation +
months of integration, depending on candidate quality.

### What the cost estimate should have said

The honest estimate for delivering a working ECP5 PCIe Gen2 PHY
upstream — under any of 1b or 1c — is **multi-quarter, not multi-sprint**.

This is exactly the kind of finding that should surface during day-1
recon, not month-2 implementation.

## Recommendation (Agent 4)

Pause coding work and **bring the scope correction back to the human +
Agent R** before committing further effort. Three actionable paths:

1. **Confirm the long-horizon path (1b or 1c) is still on-mission.**
   The cooperative's commitment to upstream contributions is real and
   load-bearing for differentiation, so a multi-quarter Agent 4 lane
   is defensible. But it should be an explicit choice, not the result
   of an under-informed estimate.
2. **Open dialogue with the LitePCIe maintainers first.** Before
   sinking effort, file an upstream issue / discussion against
   `enjoy-digital/litepcie` summarising the gap and asking for
   guidance on the preferred architecture (sub-option 1c candidate,
   datapath width, integration strategy). This is the cheapest signal.
   The maintainers' email is in their README
   (`florent [AT] enjoy-digital.fr`); they also list paid-sponsorship
   labels on related issues, so a Lattice-support sponsorship dialogue
   is also possible if the cooperative chooses to fund the work.
3. **Defer the ECP5 PHY work to post-rev-A.** Now that GbE is the rev-A
   host link, the ECP5 PHY contribution is no longer time-sensitive.
   Parking it until rev-B planning (where CertusPro-NX is the silicon
   target and the PHY landscape is different) is a defensible
   conservative choice.

Agent 4 default if no further direction lands: option 2 — open the
upstream dialogue, document the response, and wait for explicit
prioritisation before any implementation.

## Upstream link

Public scoping discussion filed as a regular issue (GitHub Discussions
is not enabled on `enjoy-digital/litepcie`, so the issue tracker was
used as the fallback public artefact, with `[discussion]` prefix in
the title to signal intent):

- `https://github.com/enjoy-digital/litepcie/issues/164`

## Resolution status

- **2026-05-05:** Scoping complete. Scope correction documented.
  Awaiting human + Agent R direction on whether to continue (and on
  which sub-option), open upstream dialogue first, or defer.
- **2026-05-06:** Outreach email sent to
  `florent@enjoy-digital.fr`. Public GitHub discussion filed at
  `https://github.com/enjoy-digital/litepcie/issues/164` (filed as an issue with
  `[discussion]` prefix because Discussions is disabled on the
  upstream repo). Awaiting maintainer response. Next checkpoint:
  2026-05-13 (one week).

Authored by Agent 4 (Open FPGA Upstream Contributions).
