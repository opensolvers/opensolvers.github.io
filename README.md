OpenSolvers explores how open-source scientific software runs on real hardware — starting with **RISC-V** boards and the tools that make that practical (EESSI, OpenBLAS, HPL, and friends). This site documents what we learn along the way.

## What we're working on

We're benchmarking **High Performance Linpack (HPL)** across several consumer RISC-V boards through the [EESSI](https://www.eessi.io/) stack. On the VisionFive 2, U74-tuned OpenBLAS lifts HPL **3.13 → 5.28 GFLOP/s** (**1.69×**). On the Orange Pi RV2, fixing an OpenBLAS RVV `gemv_n` bug turns a failing stock run into **10.53 GFLOP/s** (PASSED).

See the [HPL results overview](apps/hpl.html) for a cross-board summary.

## Boards

- **[StarFive VisionFive 2](boards/VisionFive2.html)** — JH7110 SoC, 4× SiFive U74 (`rv64gc`). EESSI HPL **3.13 → 5.28 GFLOP/s** with U74 OpenBLAS tuning.
- **[Orange Pi RV2](boards/RV2.html)** — SpaceMiT K1 SoC, 8× SpacemiT X60 (RVV). EESSI HPL **FAILED (`nan`) → 10.53 GFLOP/s** with fixed OpenBLAS.
- **[Banana Pi F3](boards/F3.html)** — SpacemiT K1 SoC, 8× SpacemiT X60. Same RVV fix as RV2; peak HPL not yet recorded.

Use the menu above to jump to a board, app, or scientific lib page.
