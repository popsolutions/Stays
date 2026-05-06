<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# bringup/ — first-power-on and validation scripts for Stays PCBs

## Purpose

Python and shell scripts that turn a freshly-fabricated PCB into a
verified working board. Coverage:

- **Power-on sequence checks** — voltage rails come up in the
  expected order; current draw is within the predicted budget.
- **JTAG access** — board enumerates via openFPGALoader or
  ecpprog on a Tigard-class adapter; bitstream load round-trip
  works.
- **DDR3 calibration** — LiteDRAM training converges on the
  socketed SO-DIMM; read/write sweep passes at the rated rate.
- **PCIe link training** — Gen2 x1 link comes up; host enumerates
  the Sail as a PCI device.
- **Inter-card link smoke** — physically present connector is
  electrically idle on rev-A, but presence and pinout are
  verified end-to-end.

## Tool list (reproducible toolchain target)

| Tool | Role | Install (Debian / Ubuntu) | Install (Arch / Manjaro) |
|------|------|---------------------------|--------------------------|
| openFPGALoader | bitstream load over JTAG / USB-Blaster / Tigard | `apt install openfpgaloader` | `pacman -S openfpgaloader` |
| ecpprog | ECP5-specific bitstream tool (alt) | from source | from AUR |
| picocom | serial console for USB-UART | `apt install picocom` | `pacman -S picocom` |
| Python ≥ 3.10 | bring-up script runtime | `apt install python3` | `pacman -S python` |
| pytest | bring-up test runner | `pip install pytest` | `pacman -S python-pytest` |
| ipmitool | board sensor reads (where IPMI present) | `apt install ipmitool` | `pacman -S ipmitool` |

The tool list is intentionally limited to packages available
through major Linux distros' default repositories so any
cooperative member, anywhere, can reproduce the bring-up
environment without manual builds.

## Test layout

Per the project's "always include tests" rule, every bring-up
script is paired with a pytest in this directory:

```
bringup/
├── README.md
├── check_<feature>.py          (the script)
├── test_check_<feature>.py     (its pytest)
└── ...
```

CI runs `pytest bringup/` on every PR (job to be added in the
follow-up PR that lands the first bring-up script).

## Licence (software)

Bring-up scripts are licensed **Apache-2.0**. Every `.py` and
`.sh` file in this directory carries the SPDX header
`# SPDX-License-Identifier: Apache-2.0`.

## See also

- [`../docs/adr/0001-fpga-target.md`](../docs/adr/0001-fpga-target.md)
  — FPGA chip and memory profile.
- [`../docs/PCB_DESIGN.md`](../docs/PCB_DESIGN.md) — power tree,
  layer stackup, routing rules.
