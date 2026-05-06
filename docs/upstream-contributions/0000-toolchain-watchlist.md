<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# Upstream toolchain watchlist — rev A (ECP5-85F + DDR3 SO-DIMM + PCIe Gen2)

This is the Agent 4 working index for the open FPGA toolchain that
ADR-001 locks for rev A. It enumerates each upstream project we depend
on, the specific integration point where we expect to interact with
the upstream codebase, and the status of any gaps we have already hit
or anticipate hitting.

This file is not a contribution log itself. Real contributions live in
sibling files following the README format
(`YYYY-MM-DD-<project>-<scope>.md`). This file points at them and tracks
what's outstanding.

## Scope discipline

Per ADR-001, rev A is **ECP5-85F + DDR3-1600 SO-DIMM single channel +
PCIe Gen2 hard IP**. Watchlist entries that fall outside that envelope
(DDR4, DDR5, CertusPro-NX, PCIe Gen3+) belong to rev B / rev C and
should not be filed against this watchlist.

## Upstream projects in the rev-A critical path

| Project | Repo | Used for | Watchlist status |
|---|---|---|---|
| yosys | https://github.com/YosysHQ/yosys | RTL synthesis (MAST → ECP5 netlist) | no entries yet |
| nextpnr-ecp5 | https://github.com/YosysHQ/nextpnr | Place & route on ECP5 fabric | no entries yet |
| prjtrellis | https://github.com/YosysHQ/prjtrellis | ECP5 bitstream database + packer | no entries yet |
| LiteDRAM | https://github.com/enjoy-digital/litedram | DDR3 controller for SO-DIMM | no entries yet |
| LitePCIe | https://github.com/enjoy-digital/litepcie | PCIe Gen2 host link via ECP5 hard IP | no entries yet |

Out of rev-A scope but tracked for rev B onward:

| Project | Used for | Status |
|---|---|---|
| Project Oxide (Glasgow / nextpnr-nexus) | CertusPro-NX successor PCB | rev B watch |

## How an entry gets created

When Agent 1 (RTL), Agent 2 (FPGA HW), or Agent 3 (Software) hits a
gap in any of the projects above:

1. They open a `cross-stream` issue in `popsolutions/Stays` with
   `stream-4` label, citing the symptom and a minimal reproducer if
   they have one.
2. Agent 4 picks the issue up:
   - Writes a contribution-log file
     `YYYY-MM-DD-<project>-<short-scope>.md` in this directory.
   - Reduces the reproducer further if needed.
   - Files the upstream issue / PR / discussion against the upstream
     project.
   - Links the upstream URL into the contribution-log file and updates
     this watchlist's status column.
3. When the upstream resolution merges, Agent 4 closes the loop:
   removes any local workaround in the consuming repo (Stays / MAST /
   Spanker) via a follow-up PR, and updates the contribution-log file
   to status "merged upstream".

## Anticipated gaps (not filed yet — speculation)

Based on community signal and the rev-A configuration, gaps we expect
to hit eventually but have **not yet observed**:

- LiteDRAM ECP5 PHY timing closure for `-8` speed grade SO-DIMM at
  1600 MT/s — historically tight at this corner; may need calibration
  recipe contribution.
- nextpnr-ecp5 routing congestion on dense designs touching the DDR3
  byte-lane region — may need a placement constraint recipe rather
  than an upstream code change.

These are tracked here as a heads-up only. They do not become entries
until we have a concrete reproducer.
