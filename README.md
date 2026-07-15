OpenSolvers explores how open-source scientific software runs on real hardware — starting with **RISC-V** boards and the tools that make that practical (EESSI, OpenBLAS, and friends). This site documents what we learn along the way.

## What we're working on

We benchmark **scientific libraries** and **applications** on consumer RISC-V boards through the [EESSI](https://www.eessi.io/) stack — from BLAS kernels up to full app runs — swapping fixed OpenBLAS builds via FlexiBLAS without rebuilding downstream code.

## What we optimise on the board

Each RISC-V board exposes several compute paths. We benchmark and tune them independently — often swapping backends at runtime via FlexiBLAS rather than rebuilding every app.

![Scalar, Vector, Specific, and GPU compute paths on the board](assets/images/compute-backends.svg)

| Path | What it is | Examples on our boards |
| ---- | ---------- | ---------------------- |
| **Scalar** | Portable C fallbacks — the correctness baseline | `OPENBLAS_CORETYPE=RISCV64_GENERIC`; VisionFive 2 before the U74 kernel |
| **Vector** | ISA vector extensions (RVV) in shared libs | OpenBLAS `RISCV64_ZVL256B` on SpaceMiT X60; the `gemv_n` bug we fixed |
| **Specific** | Silicon-tuned micro-kernels and custom units | U74 **4×4 DGEMM** on VisionFive 2; X60 **IME** (`smt.vmadot`) int8 on RV2 / F3 |
| **GPU** | Discrete or integrated accelerators | On the roadmap — not yet in our RISC-V board benchmarks |

Recent highlights on the Orange Pi RV2 (SpaceMiT X60, RVV): fixing an OpenBLAS `gemv_n` bug restores correctness across [BLAS](scientific-libs/blas.html), [LAPACK](scientific-libs/lapack.html), [ELPA](scientific-libs/elpa.html), [HPL](apps/hpl.html), and [Quantum ESPRESSO](apps/qe.html) — with patched RVV reaching **10.53 GFLOP/s** on Linpack, **1.58×** on a dense eigensolve, and **1.31×** on a 64-atom Si DFT SCF.

## Scientific libs

Library-level probes — performance *and* numerical correctness:

- **[BLAS](scientific-libs/blas.html)** — OpenBLAS improvements (U74 kernel, X60 `gemv_n` fix)
- **[DGEMM](scientific-libs/dgemm.html)** — `bench_dgemm` + `difftest` performance and correctness probes
- **[NumPy](scientific-libs/numpy.html)** — `bench_blas.py` DGEMM and `eigvalsh` through the SciPy stack
- **[LAPACK](scientific-libs/lapack.html)** — LAPACK path via NumPy `eigvalsh`
- **[ELPA](scientific-libs/elpa.html)** — dense eigensolver (CP2K / VASP class workloads)

## Apps

End-to-end application benchmarks on the same boards and EESSI toolchain:

- **[HPL](apps/hpl.html)** — High Performance Linpack; cross-board summary and A/B configs from [opensolvers/benchmarks](https://github.com/opensolvers/benchmarks)
- **[Quantum ESPRESSO](apps/qe.html)** — plane-wave DFT SCF (`pw.x`); whole-application BLAS backend A/B with per-routine timers

## Boards

- **[StarFive VisionFive 2](boards/VisionFive2.html)** — JH7110 SoC, 4× SiFive U74 (`rv64gc`). U74 OpenBLAS tuning: HPL **3.13 → 5.28 GFLOP/s**.
- **[Orange Pi RV2](boards/RV2.html)** — SpaceMiT K1, 8× X60 (RVV). Fixed OpenBLAS: HPL **FAILED (`nan`) → 10.53 GFLOP/s**; ELPA **34.81 s** (vs 54.92 s scalar).
- **[Banana Pi F3](boards/F3.html)** — same K1 / X60 SoC, **3.7 GB RAM**. HPL **FAILED (`nan`) → 11.52 GFLOP/s**; NumPy DGEMM up to **17.51 GFLOP/s** on patched RVV.

Use the menu above to jump to a board, app, or scientific lib page.
