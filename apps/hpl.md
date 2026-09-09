---
title: HPL results on RISC-V boards
description: High Performance Linpack on VisionFive 2, Orange Pi RV2, and Banana Pi F3 — OpenBLAS fixes, 0.3.34, BLIS end-to-end.
---

# HPL

**HPL** (High Performance Linpack) factors a large matrix and solves Ax=b — the TOP500-style dense LU stress test of BLAS end-to-end.

Configs + A/B: [opensolvers/benchmarks/hpl](https://github.com/opensolvers/benchmarks/tree/main/hpl). Backend story: [BLAS](../scientific-libs/blas.html). **Video:** [NaN Linpack on RISC-V](https://www.youtube.com/watch?v=W_-8cKA-CCU) · [EESSI blog](https://www.eessi.io/docs/blog/2026/07/12/risc-v-x60-openblas-hpl/).

Same `xhpl` throughout; OpenBLAS swapped via FlexiBLAS (no HPL rebuild). EESSI `2025.06-001`, GCC 14.3, HPL 2.3.

## Cross-board

| Board | Before (stock 0.3.30) | After (fixed OpenBLAS) |
| ----- | --------------------- | ---------------------- |
| [VisionFive 2](../boards/VisionFive2.html) (4× U74) | 3.13 GFLOP/s | **5.28 GFLOP/s** (**1.69×**) |
| [Orange Pi RV2](../boards/RV2.html) (8× X60) | ~8.5 GFLOP/s, **FAILED** (`nan`) | **10.53 GFLOP/s**, PASSED |
| [Banana Pi F3](../boards/F3.html) (8× X60, 3.7 GB) | 11.64 GFLOP/s, **FAILED** (`nan`) | **11.52 GFLOP/s**, PASSED |

X60 stock RVV looked “fast” but residual was `nan` — broken `gemv_n` ([easyconfigs#26444](https://github.com/easybuilders/easybuild-easyconfigs/pull/26444)).

## Orange Pi RV2

| Config | Result |
| ------ | ------ |
| Stock RVV N=8000 | ~8.5 GFLOP/s, **FAILED** |
| Fixed RVV peak (N=20000, 2×4) | **10.53** GFLOP/s, PASSED |
| Scalar vs patched RVV (N=8000) | 6.41 → **11.55** (**1.80×**) |
| OpenBLAS **0.3.34** (N=8000 / N=20000) | **11.04** / **10.97** vs patched 0.3.30 |

Prefer a **2×4** grid over 1×8 for peak. Large N is RAM-tight on 8 GB. Harness: [`run-hpl-ab.sh`](https://github.com/opensolvers/benchmarks/blob/main/hpl/run-hpl-ab.sh), [`run-hpl-034.sh`](https://github.com/opensolvers/benchmarks/blob/main/hpl/run-hpl-034.sh).

GCC X60 mtune is a separate axis ([GCC](../scientific-libs/gcc.html)) — modest HPL **+0.8%** on 15.2.

## HPL on BLIS

Dedicated `xhpl` + static RVV `libblis.a` (no FlexiBLAS on RV2). Correctness holds; square-DGEMM wins do not transfer:

| Config | BLIS | vs OpenBLAS-RVV |
| ------ | ---: | --------------: |
| N=8000, 1×8 | 4.02 GFLOP/s PASSED | **0.35×** |
| N=20000, 2×4 | 5.57 GFLOP/s PASSED | **0.53×** |
| N=25600 best (2×4) | **5.87** PASSED | — |

HPL is panel-heavy (`dtrsm` / level-2), not the large square DGEMM where [BLIS](../scientific-libs/blis.html) wins ~1.3× single-thread.

## Other boards

- **VisionFive 2** — U74-tuned OpenBLAS lifts HPL **1.69×** ([EESSI/docs#818](https://github.com/EESSI/docs/pull/818)).
- **BPI-F3** — same K1 fix; only N=8000 fits in 3.7 GB; patched RVV **1.77×** scalar.
