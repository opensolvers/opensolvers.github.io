---
title: scikit-learn on SpaceMiT X60
description: scikit-learn 1.4 FlexiBLAS A/B on Orange Pi RV2 — PCA 1.22×, Ridge 1.90× vs scalar with patched RVV OpenBLAS.
---

# scikit-learn

[scikit-learn](https://scikit-learn.org/) estimators dispatch into NumPy → FlexiBLAS. On the Orange Pi RV2 we A/B OpenBLAS backends under an unchanged `scikit-learn` install — same hub as [NumPy](numpy.html) and [Armadillo](armadillo.html).

Benchmark source: [opensolvers/benchmarks/sklearn](https://github.com/opensolvers/benchmarks/tree/main/sklearn). Module: `scikit-learn/1.4.0-gfbf-2023b` on EESSI `riscv.eessi.io` **20240402**.

> **Change one variable.** Same estimators / data; only FlexiBLAS OpenBLAS differs.

> **Bottom line:** patched RVV vs scalar — PCA **1.22×**, Ridge **1.90×**. Checksums match across tags.

---

## Kernels

| Kernel | Call | Notes |
| ------ | ---- | ----- |
| PCA | `PCA(svd_solver="full").fit_transform` | dense SVD / GEMM |
| Ridge | `Ridge(solver="cholesky").fit` | LAPACK / BLAS |

---

## Results (2026-08-22)

8 threads, `N=8000`, `D=512`, `K=64`.

| Tag | PCA s | Ridge s | Wall s |
| --- | ----: | ------: | -----: |
| scalar (`RISCV64_GENERIC`) | 8.33 | 0.99 | 25.91 |
| stock | 8.50 | 0.99 | 21.50 |
| patched RVV | **6.80** | **0.52** | 19.20 |

Related: [NumPy](numpy.html), [BLAS](blas.html).

**Measured:** 2026-08-22 on Orange Pi RV2.
