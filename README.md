<!--
SPDX-License-Identifier: CC-BY-SA-4.0
Copyright (c) 2026 PopSolutions Cooperative
-->

# Stays — PopSolutions Sails FPGA-side rigging

> *In a tall ship, the **stays** are the rigging that holds the mast in
> place. A strong mast with weak stays cannot move the ship forward.*

**Stays** is the FPGA-side counterpart to [`MAST`](https://github.com/popsolutions/MAST).
Where MAST holds the shared RTL trunk that becomes silicon in the long
run, Stays holds **everything that turns that RTL into a real board you
can hold in your hand right now**:

- **Custom FPGA board designs** (KiCad schematics, layout, gerbers, BOMs)
- **Bring-up scripts** for first-power-on, JTAG access, DDR3 calibration,
  PCIe link training
- **Upstream contributions** back to the open FPGA toolchain community
  (Project Oxide, prjtrellis, LiteDRAM, LitePCIe, nextpnr-ecp5, yosys) —
  tracked here as a first-class project deliverable
- **Inter-card connector specs and validation** (per the multi-card
  parallelism requirement)

The current bootstrap target is a custom PCB hosting **Lattice ECP5-85F
+ DDR3 SO-DIMM (acoplável)**. See
[`docs/adr/0001-fpga-target.md`](docs/adr/0001-fpga-target.md).

## Why a separate repo from MAST

MAST is the silicon-ready RTL: cycle-accurate, tape-out-bound, frozen
at every Sail's tape-out tag. Stays is the *journey* — the FPGA boards
that let us validate MAST RTL on real hardware right now, the bring-up
log of every prototype rev, the upstream patches we landed in open FPGA
tooling along the way.

In nautical terms: MAST is the mast itself; Stays is the rigging that
keeps it standing and lets the ship move forward. Different lifecycle,
different cadence, different audience for contributions. They live
together in the same fleet but in separate repos.

## Mission alignment

Per the project mission ([memory entry][mission]): we use AI to reduce
the technological gap, especially in the Global South. **Open-toolchain
FPGAs are the bridge** between open-source RTL (which we can write today)
and open-source silicon (which we can tape out years from now).

The open FPGA ecosystem is itself underdeveloped because closed players
hoard their tooling. Every bug we find, every fix we upstream to Project
Oxide / prjtrellis / nextpnr / LiteDRAM, every recipe for a working
DDR controller config we publish — that **is the contribution**, not
just our own boards.

## License

- Hardware contributions (KiCad schematics, layout, gerbers, BOMs):
  CERN-OHL-S v2 (strongly reciprocal). Commercial license via the
  cooperative.
- Software contributions (bring-up scripts, drivers, build glue):
  Apache 2.0.
- Documentation: CC-BY-SA 4.0.
- Upstream contributions to other open projects: licensed per the
  upstream project's terms (we contribute, we don't relicense).

## Layout

```
Stays/
├── README.md                   (this file)
├── LICENSE                     (CERN-OHL-S v2)
├── docs/
│   ├── adr/                    (irreversible architectural decisions)
│   │   └── 0001-fpga-target.md
│   ├── PCB_DESIGN.md           (process notes, layer stackup, etc.)
│   └── upstream-contributions/ (changelog of upstream patches)
├── kicad/                      (KiCad project files)
├── bringup/                    (Python / shell scripts for first-light)
└── .github/workflows/          (CI: KiCad ERC + DRC, doc lint)
```

## Status

Bootstrap (2026-05-05). No PCB committed yet; ADR-001 just landed
locking the target FPGA chip and memory profile.

## Related repos

- [`popsolutions/MAST`](https://github.com/popsolutions/MAST) — RTL trunk
- [`popsolutions/InnerJib7EA`](https://github.com/popsolutions/InnerJib7EA) — POPC_16A first product (silicon target)

[mission]: # "see the project's private memory; framed by the founding member 2026-05-05"
