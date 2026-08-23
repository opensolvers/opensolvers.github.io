---
title: Armadillo on SpaceMiT X60
description: Armadillo 12.8 FlexiBLAS A/B on Orange Pi RV2 — DGEMM 1.82×, eig_sym 1.63× vs scalar with patched RVV OpenBLAS.
---

# Armadillo

[Armadillo](https://arma.sourceforge.net/) is a C++ template linear-algebra library (matrix ops → BLAS/LAPACK). On the Orange Pi RV2 we swap the OpenBLAS backend under an unchanged Armadillo binary via FlexiBLAS — same pattern as [NumPy](numpy.html).

Benchmark source: [opensolvers/benchmarks/armadillo](https://github.com/opensolvers/benchmarks/tree/main/armadillo). Module: `Armadillo/12.8.0-foss-2023b` on EESSI `riscv.eessi.io` **20240402**.

> **Change one variable.** Same binary / problem sizes; only FlexiBLAS OpenBLAS (scalar vs stock vs patched RVV) differs.

> **Bottom line:** patched RVV vs scalar — DGEMM **1.82×**, `eig_sym` **1.63×**, wall **1.66×**. Stock ≈ scalar at these sizes. All finite.

---

## Kernels

| Kernel | Call | Backend | Metric |
| ------ | ---- | ------- | ------ |
| DGEMM | `C = A * B` | `dgemm` | GFLOP/s |
| EIG | `eig_sym(S)` | `dsyev*` | seconds |

---

## Results (2026-08-22)

8 threads, `N_dgemm=2048`, `N_eig=1024`.

| Tag | DGEMM GFLOP/s | EIG s | Wall s |
| --- | ------------: | ----: | -----: |
| scalar (`RISCV64_GENERIC`) | 4.55 | 1.42 | 18.75 |
| stock | 4.66 | 1.39 | 18.34 |
| patched RVV | **8.27** | **0.87** | 11.30 |

Related: [NumPy](numpy.html), [BLAS](blas.html).

**Measured:** 2026-08-22 on Orange Pi RV2.
