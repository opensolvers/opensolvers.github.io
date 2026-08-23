---
title: PETSc on SpaceMiT X60
description: PETSc 3.24 on Orange Pi RV2 — FlexiBLAS A/Bs (dense MatMult ~1.70×; MUMPS 3D n=80 ~1.5×; stock RVV NaN on dense/direct) and hand RVV SpMV (stencil ~3.6× vs MatMult).
---

# PETSc

[PETSc](https://petsc.org/) (Portable, Extensible Toolkit for Scientific Computation) is a parallel library for sparse and dense linear algebra, Krylov solvers, and preconditioners — the backbone of many CFD, structural, and multiphysics codes. On the Orange Pi RV2 (SpaceMiT X60, RVV 1.0, VLEN=256) we probe two axes: **FlexiBLAS** backend A/Bs under one unchanged PETSc binary, and a **hand RVV SpMV** probe against PETSc `MatMult`.

Benchmark source: [opensolvers/benchmarks/petsc](https://github.com/opensolvers/benchmarks/tree/main/petsc).

> **Change one variable (FlexiBLAS).** Hold problem + solver fixed; swap only the backend (patched OpenBLAS vs stock RVV vs scalar). Check finite residuals before trusting wall time.

> **Bottom line:** sparse Jacobi-CG is a weak BLAS lever (**~1.06×**). Dense MatMult shows patched RVV **~1.70×** vs scalar — and **stock RVV NaNs** (same OpenBLAS `gemv_n` class as [HPL](../apps/hpl.html) / [ELPA](elpa.html)). MUMPS 3D stays flat at small fronts; at **n=80** (512k dofs) both RVV backends beat scalar ~**1.5×**. Hand RVV on generic CSR SpMV ≈ **no win**; a structure-aware 5-point stencil is **~3.6×** vs PETSc `MatMult`.

Related: [BLAS](blas.html), [EESSI X60 blog](https://www.eessi.io/docs/blog/2026/07/12/risc-v-x60-openblas-hpl/).

---

## Setup

| Piece | Value |
| ----- | ----- |
| Board | Orange Pi RV2 · 8× X60 @ ~1.6 GHz |
| PETSc | overlay `PETSc/3.24.0-foss-2025b` (+ SuiteSparse, Hypre, SuperLU_DIST, MUMPS, PnetCDF) |
| Stack | EESSI `2025.06-001` `foss/2025b` · FlexiBLAS 3.4.5 |
| FlexiBLAS A/Bs | 8 threads · best of 3 |
| SpMV A/B | 1 thread · best of 20 · `-march=rv64gcv` |

Harness: `petsc_ksp_bench.c`, `petsc_dense_bench.c`, `petsc_direct_bench.c`, `petsc_spmv_rvv_bench.c` + `run-petsc-*-ab.sh`.

---

## Hand RVV SpMV vs PETSc `MatMult`

2D 5-point Laplacian, **n=800** (640k dofs). All kernels bit-match PETSc (`max_abs=0`).

| Kernel | BEST WALL | vs PETSc |
| ------ | --------: | -------: |
| PETSc `MatMult` | 0.0417 s | 1.00× |
| CSR scalar | 0.0494 s | 0.84× |
| CSR RVV (gather) | 0.0489 s | 0.85× |
| stencil5 scalar | 0.0138 s | **3.03×** |
| stencil5 RVV | **0.0116 s** | **3.59×** |

1. **Generic CSR RVV does not help** this AIJ (≈5 nnz/row): gather + short vectors ≈ scalar CSR, and both trail PETSc’s tuned `MatMult` slightly.
2. **Structure-aware stencil** (same operator, contiguous loads) is the real PETSc-local win: **~3.6×** vs `MatMult`, with a further **~1.19×** from hand RVV over scalar stencil.
3. For upstream PETSc on RVV: invest in **MFD / stencil / BAIJ** kernels, not a naive CSR `vluxei` SpMV for PDE-style matrices.

Log: [`petsc-spmv-rvv-ab-20260815T081626Z.txt`](https://github.com/opensolvers/benchmarks/blob/main/petsc/results/petsc-spmv-rvv-ab-20260815T081626Z.txt).

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

### MUMPS 3D scale sweep (2026-08-22)

`petsc_direct_bench`, 3 reps, 8 threads. Summary: [`mumps-3d-scale-ab-20260822.txt`](https://github.com/opensolvers/benchmarks/blob/main/petsc/results/mumps-3d-scale-ab-20260822.txt).

| 3D n | dofs | scalar BEST | stock RVV | patched RVV | patched / scalar |
| ---: | ---: | ----------: | --------: | ----------: | ---------------: |
| 40 | 64k | 0.652 s | 0.674 s | 0.669 s | ~1.0× (flat) |
| 60 | 216k | 2.772 s | 2.874 s | **2.681 s** | ~1.03× |
| 80 | 512k | 12.585 s | **7.987 s** | 8.470 s | **1.49×** |

At n=80 both RVV backends beat scalar (~**1.5×**); stock is slightly ahead of patched here. All runs `finite=1`, identical residuals. RVV wins once frontal dense blocks grow large enough.

---

## Reading

1. **Dense PETSc paths** expose FlexiBLAS clearly: patched RVV wins; stock RVV corrupts.
2. **SuperLU_DIST / UMFPACK** need the patched OpenBLAS for correctness — same failure class as HPL / ELPA / QE.
3. **MUMPS** stays finite on stock at small sizes with little/no patched speedup; at **3D n=80** RVV is ~**1.5×** scalar.
4. **Jacobi-CG AIJ** remains a weak BLAS A/B (~1.06×).
5. **Hand RVV SpMV:** CSR gather is a dead end on short-row PDE matrices; stencil / structure-aware kernels are where RVV pays.

**Measured:** 2026-08-14 (FlexiBLAS) / 2026-08-15 (SpMV) / 2026-08-22 (MUMPS scale) on Orange Pi RV2. Logs in [benchmarks/petsc/results/](https://github.com/opensolvers/benchmarks/tree/main/petsc/results).
