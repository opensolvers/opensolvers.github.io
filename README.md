OpenSolvers explores how open-source scientific software runs on real hardware — starting with **RISC-V** boards and the tools that make that practical (EESSI, OpenBLAS, and friends). This site documents what we learn along the way.

## What we're working on

We benchmark **scientific libraries** and **applications** on consumer RISC-V boards through the [EESSI](https://www.eessi.io/) stack — from BLAS kernels up to full app runs — swapping fixed OpenBLAS builds via FlexiBLAS without rebuilding downstream code.

Recent highlights on the Orange Pi RV2 (SpaceMiT X60, RVV): fixing an OpenBLAS `gemv_n` bug restores correctness across [BLAS](scientific-libs/blas.html), [LAPACK](scientific-libs/lapack.html), [ELPA](scientific-libs/elpa.html), and [HPL](apps/hpl.html) — with patched RVV reaching **10.53 GFLOP/s** on Linpack and **1.58×** on a dense eigensolve.

## Scientific libs

Library-level probes — performance *and* numerical correctness:

- **[BLAS](scientific-libs/blas.html)** — OpenBLAS improvements (U74 tuned kernel, X60 RVV `gemv_n` fix) and `difftest` / `bench_dgemm` verification
- **[LAPACK](scientific-libs/lapack.html)** — NumPy `dgemm` and `eigvalsh` as a real SciPy-stack probe
- **[ELPA](scientific-libs/elpa.html)** — dense eigensolver mixing BLAS-2 and BLAS-3 (CP2K / VASP class workloads)

## Apps

End-to-end application benchmarks on the same boards and EESSI toolchain:

- **[HPL](apps/hpl.html)** — High Performance Linpack; cross-board summary and A/B configs from [opensolvers/benchmarks](https://github.com/opensolvers/benchmarks)

## Boards

- **[StarFive VisionFive 2](boards/VisionFive2.html)** — JH7110 SoC, 4× SiFive U74 (`rv64gc`). U74 OpenBLAS tuning: HPL **3.13 → 5.28 GFLOP/s**.
- **[Orange Pi RV2](boards/RV2.html)** — SpaceMiT K1, 8× X60 (RVV). Fixed OpenBLAS: HPL **FAILED (`nan`) → 10.53 GFLOP/s**; ELPA **34.81 s** (vs 54.92 s scalar).
- **[Banana Pi F3](boards/F3.html)** — same K1 / X60 SoC; same fixes apply, peak results not yet recorded.

Use the menu above to jump to a board, app, or scientific lib page.
