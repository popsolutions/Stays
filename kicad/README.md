<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# kicad/ — KiCad project files for PopSolutions Sails PCBs

This directory holds the KiCad project files for every PCB produced
by the Stays repo. One subdirectory per Sail codename + revision
letter (per `docs/PCB_DESIGN.md`):

```
kicad/
└── <sail-codename>-rev-<letter>/
    ├── <project>.kicad_pro
    ├── <project>.kicad_sch
    ├── <project>.kicad_pcb
    └── README.md
```

## Naming convention

- Lowercase Sail codename (e.g., `innerjib7ea`).
- Revision letter starts at `a` and increments per PCB rev.
- Project filename matches the directory name without the
  `-rev-<letter>` suffix (e.g.,
  `kicad/innerjib7ea-rev-a/innerjib7ea.kicad_pro`).

## KiCad version target

**KiCad 8 or later.** The CI job (`.github/workflows/ci.yml`,
`kicad-erc-drc`) installs the Ubuntu-packaged KiCad and runs
`kicad-cli sch erc` and `kicad-cli pcb drc` on every `.kicad_sch`
and `.kicad_pcb` file in this tree on every PR. The job exits
cleanly when no KiCad files are present yet.

## Licence (hardware design files)

KiCad design files in this tree are licensed
**CERN-OHL-S v2** (strongly reciprocal). Every CAD-file commit
carries the appropriate SPDX header in its commit message; where
the KiCad file format permits, an embedded SPDX line is included
inside the file.

A commercial licence is available from the cooperative.

## See also

- [`docs/adr/0001-fpga-target.md`](../docs/adr/0001-fpga-target.md)
  — locks the rev-A FPGA chip and memory profile.
- [`docs/PCB_DESIGN.md`](../docs/PCB_DESIGN.md) — running notebook
  for PCB design conventions, layer stackup, routing rules, power
  tree.
