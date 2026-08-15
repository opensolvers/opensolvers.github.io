---
title: PETSc on SpaceMiT X60
description: PETSc 3.24 FlexiBLAS A/Bs on Orange Pi RV2 — sparse Jacobi-CG, dense MatMult/CG, and MUMPS / SuperLU_DIST / UMFPACK; stock RVV NaN on dense and some direct solvers.
---

# PETSc

[PETSc](https://petsc.org/) (Portable, Extensible Toolkit for Scientific Computation) is a parallel library for sparse and dense linear algebra, Krylov solvers, and preconditioners — the backbone of many CFD, structural, and multiphysics codes. On the Orange Pi RV2 (SpaceMiT X60, RVV 1.0, VLEN=256) we A/B the **BLAS / LAPACK backend** through FlexiBLAS under one unchanged PETSc binary.

Benchmark source: [opensolvers/benchmarks/petsc](https://github.com/opensolvers/benchmarks/tree/petsc-flexiblas-ab/petsc).

> **Change one variable.** Hold problem + solver fixed; swap only the FlexiBLAS backend (patched OpenBLAS vs stock RVV vs scalar). Check finite residuals before trusting wall time.

> **Bottom line:** sparse Jacobi-CG is a weak BLAS lever (**~1.06×**). Dense MatMult shows patched RVV **~1.70×** vs scalar — and **stock RVV NaNs** (same OpenBLAS `gemv_n` class as [HPL](../apps/hpl.html) / [ELPA](elpa.html)). SuperLU_DIST and UMFPACK need the patched backend for correctness; MUMPS stayed finite here but showed no patched speedup at these sizes.

Related: [BLAS](blas.html), [EESSI X60 blog](https://www.eessi.io/docs/blog/2026/07/12/risc-v-x60-openblas-hpl/).

---

## Setup

| Piece | Value |
| ----- | ----- |
| Board | Orange Pi RV2 · 8× X60 @ ~1.6 GHz |
| PETSc | overlay `PETSc/3.24.0-foss-2025b` (+ SuiteSparse, Hypre, SuperLU_DIST, MUMPS, PnetCDF) |
| Stack | EESSI `2025.06-001` `foss/2025b` · FlexiBLAS 3.4.5 |
| Threads | 8 · best of 3 |

Harness: `petsc_ksp_bench.c`, `petsc_dense_bench.c`, `petsc_direct_bench.c` + `run-petsc-*-ab.sh`.

---

## Sparse Jacobi-CG (weak BLAS lever)

2D Laplacian **n=400** (160k dofs), Jacobi + CG.

| Backend | BEST WALL | its | finite |
| ------- | --------: | --: | -----: |
| scalar | **11.024 s** | 734 | ✓ |
| stock RVV | **10.571 s** (~1.04×) | 734 | ✓ |
| patched RVV | **10.374 s** (~1.06×) | 734 | ✓ |

AIJ MatMult / Jacobi dominate; FlexiBLAS barely moves the needle. Use dense or direct for backend validation.

---

## Dense MatMult / CG

### Dense MatMult (`MATDENSE` n=2048)

| Backend | BEST WALL | GF/s | finite |
| ------- | --------: | ---: | -----: |
| scalar | 0.01020 s | 0.823 | ✓ |
| stock RVV | 0.0280 s | 0.300 | **NaN** (`\|y\|=nan`) |
| patched RVV | **0.00600 s** | **1.397** | ✓ |

Patched / scalar ≈ **1.70×**. Stock hits the known OpenBLAS `gemv_n` NaN bug.

### Dense CG (n=1024, `PCNONE`)

| Backend | BEST WALL | its | finite |
| ------- | --------: | --: | -----: |
| scalar | 0.0232 s | 3 | ✓ |
| stock RVV | 0.0074 s | 1 | **diverged / NaN** |
| patched RVV | **0.00660 s** | 3 | ✓ |

Patched / scalar ≈ **3.5×** (tiny iteration count; still shows BLAS on the MatMult path).

---

## Sparse direct LU

| Solver | Problem | scalar | stock RVV | patched RVV |
| ------ | ------- | -----: | --------: | ----------: |
| **MUMPS** | 2D n=200 (40k dofs) | **0.097 s** ✓ | 0.112 s ✓ | 0.112 s ✓ |
| **MUMPS** | 3D n=40 (64k dofs) | **0.586 s** ✓ | 0.635 s ✓ | 0.633 s ✓ |
| **SuperLU_DIST** | 2D n=200 | **0.030 s** ✓ | 0.032 s **NaN** | 0.032 s ✓ |
| **UMFPACK** | 2D n=200 | **0.038 s** ✓ | 0.178 s **NaN** | 0.038 s ✓ |

---

## Reading

1. **Dense PETSc paths** expose FlexiBLAS clearly: patched RVV wins; stock RVV corrupts.
2. **SuperLU_DIST / UMFPACK** need the patched OpenBLAS for correctness — same failure class as HPL / ELPA / QE.
3. **MUMPS** stayed finite on stock at these sizes but showed **no** patched speedup (analysis / ordering / smaller dense fronts dominate). Larger 3D problems would be the next lever for frontal BLAS-3.
4. **Jacobi-CG AIJ** remains a weak BLAS A/B (~1.06×).

**Measured:** 2026-08-14 on Orange Pi RV2. Logs in [benchmarks/petsc/results/](https://github.com/opensolvers/benchmarks/tree/petsc-flexiblas-ab/petsc/results).
