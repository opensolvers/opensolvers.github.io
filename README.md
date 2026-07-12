OpenSolvers explores how open-source scientific software runs on real hardware — starting with **RISC-V** boards and the tools that make that practical (EESSI, OpenBLAS, HPL, and friends). This site documents what we learn along the way.

## What we're working on

We're benchmarking **High Performance Linpack (HPL)** across several consumer RISC-V boards, comparing stock and architecture-tuned builds through the [EESSI](https://www.eessi.io/) stack. On the VisionFive 2, a SiFive U74-tuned OpenBLAS lifts HPL from **3.13** to **5.28 GFLOP/s** (**1.69×**) — end-to-end via EESSI and EasyBuild.

See the [HPL results overview](apps/hpl.html) for a cross-board summary.

## Boards

- **[StarFive VisionFive 2](boards/VisionFive2.html)** — JH7110 SoC, 4× SiFive U74 (`rv64gc`). EESSI HPL **3.13 → 5.28 GFLOP/s** with U74 OpenBLAS tuning.
- **[OrangePi RV2](boards/RV2.html)** — Ky X1 SoC, 8× SpacemiT X60. Native-arch HPL **7.38 GFLOP/s** (EESSI: DNF, RVV bug in OpenBLAS 0.3.30).
- **[BananaPi F3](boards/F3.html)** — SpacemiT K1 SoC, 8× SpacemiT X60. HPL benchmarking in progress (EESSI: DNF, RVV bug in OpenBLAS 0.3.30).

Use the menu above to jump to a board or app page.
