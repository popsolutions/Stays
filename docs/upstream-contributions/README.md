<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# Upstream contributions log

This directory records every contribution PopSolutions Stays makes back
to the open FPGA / open hardware toolchain ecosystem. Each contribution
is a first-class deliverable of the project alongside our own boards.

Format: one Markdown file per contribution, named by upstream project
and date:

```
docs/upstream-contributions/
├── 2026-XX-XX-litedram-ddr3-ecp5-config.md
├── 2026-XX-XX-nextpnr-ecp5-routing-bug.md
├── 2026-XX-XX-prjtrellis-bitstream-fix.md
└── ...
```

Each file should record:

1. The upstream project (LiteDRAM / Project Oxide / nextpnr-ecp5 /
   yosys / prjtrellis / etc.)
2. The bug or feature gap we hit
3. Our local workaround (if any)
4. The link to the upstream issue / PR / discussion
5. Resolution status

Even bug *reports* count — well-written reports save other adopters
time and give upstream maintainers signal that real-world consumers
care about the issue.

This is not optional housekeeping. Per the project's mission framing,
**advancing the open FPGA ecosystem is on equal footing with shipping
our own hardware**.
