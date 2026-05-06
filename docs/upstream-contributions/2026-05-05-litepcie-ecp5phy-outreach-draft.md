<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# 2026-05-05 — LitePCIe ECP5 PHY upstream outreach (DRAFT)

**Status:** Draft — for human review before sending.

## What this is

This document is a **draft email** for Marcos Méndez Quintero
(`m@pop.coop`) to send personally from his mail client to Florent
Kermarrec (LitePCIe maintainer, `florent@enjoy-digital.fr`).

It is **not** a public communication from the cooperative, **not** an
issue, **not** a PR. The cooperative's public artefacts on this topic
are:

- `docs/upstream-contributions/0001-rev-a-known-upstream-issues.md`
  (the rev-A survey, on `main`).
- `docs/upstream-contributions/2026-05-05-litepcie-ecp5phy.md` (the
  contribution log entry in scoping state, currently on PR #17).
- popsolutions/MAST#13 (the cross-stream issue documenting the
  decision).

The email below is calibration — we want to know whether and how
upstream would like a contribution before sinking multi-quarter
effort. It commits the cooperative to no timeline, no scope, and no
delivery.

## How this gets sent

1. Agent 4 drafts (this PR).
2. Agent R reviews the draft like any other PR — same `review-pending`
   + `stream-4` discipline.
3. Marcos reads the merged content, copy-pastes into his mail client,
   adapts tone if he wants, sends from `m@pop.coop`.
4. Once Marcos has sent it, Agent 4 updates
   `2026-05-05-litepcie-ecp5phy.md` from `scoping` →
   `awaiting-upstream-feedback` with the send date in a follow-up PR.
5. When Florent replies (or doesn't, after a reasonable interval),
   Marcos reports the response and Agent 4 updates the contribution
   log to `maintainer-response-{accepted|rejected|deferred}` plus
   next-step derivation.

Agent 4 does **not** send mail. Agent 4 does **not** use any
sock-puppet account. Agent 4 does **not** post the email content
anywhere public until after Florent has replied (the contribution log
will summarise the exchange respectfully, not paste the raw email).

## Email draft

---

**From:** Marcos Méndez Quintero `<m@pop.coop>`
**To:** Florent Kermarrec `<florent@enjoy-digital.fr>`
**Subject:** LitePCIe + Lattice ECP5 — scoping question from a small open-hardware cooperative

Hi Florent,

I run **PopSolutions Cooperative** — a small open-hardware project
designing FPGA-based AI inference boards aimed at making model
execution accessible outside the regions that can buy a GPU at sticker
price. Our rev-A board uses a Lattice ECP5-85F with a DDR3-1600
SO-DIMM and an end-to-end open toolchain (yosys + nextpnr-ecp5 +
prjtrellis + LiteDRAM). Public, BSD-style work; not a proprietary
fork trying to commoditise your tooling.

I'm writing to you directly rather than opening a GitHub issue
because (a) the LitePCIe README points to this address for support
discussions, (b) what I have is an architectural question, not a bug
report, and (c) I'd rather calibrate expectations privately before
the cooperative spends real time on a contribution you might not want.

### What we found

While scoping our rev-A host link, we did a one-day audit of LitePCIe:

- `litepcie/phy/` ships PHY modules for Cyclone V, Gowin GW5A, Lattice
  CertusPro-NX, and three Xilinx variants. There is no
  `ecp5pciephy.py`.
- A search of the LitePCIe issue tracker and PR history for
  `lattice OR ecp5` returns zero results — open, closed, or merged.
- The README's "Possible improvements" list includes "add Lattice
  support", which is what motivated us to dig further.
- Reading `phy/lfcpnxpciephy.py` end-to-end, we noticed it wraps a
  Lattice-generated Verilog IP blob downloaded at build time. That
  pattern works for CertusPro-NX because the silicon ships an
  integrated PCIe controller. ECP5 is different: it exposes the PCS /
  SerDes hard block (DCU) only — the PIPE-to-TLP stack (LTSSM, DLL,
  TLP framing, flow-control credits) has to be soft-implemented or
  ported. The closed Diamond toolchain has a soft IP for this; we
  haven't been able to find a production-mature open equivalent.

That makes a hypothetical `ecp5pciephy.py` substantially more work
than the CertusPro-NX file would suggest at first glance. Our honest
internal estimate is multi-quarter, not multi-sprint.

(Our public scoping notes, in case the file references are useful:
`https://github.com/popsolutions/Stays/blob/main/docs/upstream-contributions/0001-rev-a-known-upstream-issues.md`
— rev-A toolchain survey that surfaced the gap.)

### What I'd like to ask you

No specific commitment expected from this email — these are
calibration questions:

1. **Is our diagnosis correct?** Specifically, that ECP5 has only a
   PCS hard block and that any LitePCIe ECP5 PHY would need a soft
   PCIe controller stack on top, or alternatively a wrapper around
   the closed Diamond IP?
2. **Has the project considered ECP5 support before?** If so, where
   did the consideration stop, and what would change to restart it?
3. **What scope of initial PR would you find acceptable?** A bare
   PHY-layer module that assumes downstream consumers handle
   DLL/TL? A full stack on first PR? Something stacked over multiple
   PRs (PHY → DLL → TL across releases)?
4. **Is sponsorship the right channel for work of this scope?** I
   noticed `sponsor-welcome` labels on related issues (e.g. #122).
   The cooperative has limited funding but takes the question
   seriously — if a paid scoping engagement is the right way to start
   rather than community-contribution, we'd like to know.

A response of "we don't think this fits LitePCIe upstream" or "we'd
only want this work under a sponsorship contract" or "this is too far
from our priorities to give input on" is equally legitimate and
equally welcome. I'd rather know now than discover it after a PR.

In parallel with this question, our rev-A board is switching its host
link to GbE via LiteEth, so PopSolutions is not blocked on an answer
— this contribution is a forward-looking gift on its own honest
timeline, not a tactical necessity.

Thank you for the time, and for LitePCIe in general — the codebase
read cleanly enough that a one-day external audit could form a
reasonable opinion, which speaks to the project's quality.

Best regards,
Marcos Méndez Quintero
PopSolutions Cooperative
`m@pop.coop`
`https://github.com/popsolutions`

---

## Notes for Marcos before sending

- Replace any phrasing that doesn't sound like you. The above
  prioritises calibration over personality; your real voice will read
  better than a polished draft.
- Verify the public link in section "Our public scoping notes" still
  resolves to the survey doc on `main` at the moment you send (the
  doc is on `main` as of 2026-05-05 via PR #3).
- If you want a public artefact later, the contribution-log PR
  (popsolutions/Stays#17) is where the response gets summarised,
  respectfully and without quoting Florent's reply verbatim unless he
  authorises it.
- If you adapt the subject line, keep three signals: ECP5/Lattice,
  scoping/exploratory, small-cooperative (so it stands apart from
  vendor outreach in his inbox).

Authored by Agent 4 (Open FPGA Upstream Contributions).
