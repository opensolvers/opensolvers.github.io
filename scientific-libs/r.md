---
title: R on SpaceMiT X60
description: R 4.4 FlexiBLAS A/B on Orange Pi RV2 — GEMM 1.80×, eigen 1.45× vs scalar with patched RVV OpenBLAS.
---

# R

[R](https://www.r-project.org/) matrix ops (`%*%`, `eigen`) dispatch into the same FlexiBLAS stack as [NumPy](numpy.html) and [Armadillo](armadillo.html). On the Orange Pi RV2 we swap OpenBLAS backends under an unchanged `R` binary.

Benchmark source: [opensolvers/benchmarks/r](https://github.com/opensolvers/benchmarks/tree/main/r). Module: `R/4.4.1-gfbf-2023b` on EESSI `riscv.eessi.io` **20240402**.

> **Change one variable.** Same Rscript / sizes; only FlexiBLAS OpenBLAS differs.

> **Bottom line:** patched RVV vs scalar — GEMM **1.80×**, EIGEN **1.45×**, wall **1.34×**. Eigenvalue checksum matches across tags.

---

## Kernels

| Kernel | Call | Backend | Metric |
| ------ | ---- | ------- | ------ |
| GEMM | `A %*% B` | `dgemm` | GFLOP/s |
| EIGEN | `eigen(S, symmetric=TRUE)` | `dsyev*` | seconds |

---

## Results (2026-08-22)

8 threads, `N_dgemm=2048`, `N_eig=1024`.

| Tag | GEMM GFLOP/s | EIGEN s | Wall s |
| --- | -----------: | ------: | -----: |
| scalar (`RISCV64_GENERIC`) | 4.47 | 1.72 | 29.41 |
| stock | 4.42 | 1.62 | 29.38 |
| patched RVV | **8.03** | **1.19** | 21.94 |

Related: [NumPy](numpy.html), [BLAS](blas.html).

**Measured:** 2026-08-22 on Orange Pi RV2.
