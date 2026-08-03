---
title: OpenFOAM motorBike on SpaceMiT X60
description: OpenFOAM v2506 simpleFoam motorBike on Orange Pi RV2 — GCC auto-vec and hand RVV Amul / Gauss–Seidel A/Bs (sparse gather regressions).
---

# OpenFOAM

[OpenFOAM](https://www.openfoam.com/) **v2506** on the Orange Pi RV2 (SpaceMiT K1 / X60, RVV 1.0) — measured A/Bs on the canonical **motorBike** tutorial (`simpleFoam`, 4 MPI ranks, existing mesh, `endTime=50`).

Benchmark source: [opensolvers/benchmarks/openfoam](https://github.com/opensolvers/benchmarks/tree/main/openfoam).

> **Bottom line:** neither GCC auto-vec nor hand RVV Amul / Gauss–Seidel gather paths win on this board. Sparse gather on short LDU/CSR rows regresses; GS still dominates wall time and stays hard to SIMD without recolouring.

Contrast contiguous auto-vec wins in [waLBerla](walberla.html) (HeatEquation / UniformGrid collide).

---

## Setup

| Piece | Value |
| ----- | ----- |
| Board | Orange Pi RV2 (~7.7 GiB) |
| Stack | EESSI `2025.06-001` + overlay OpenFOAM `v2506-foss-2025b-noPV` |
| Case | motorBike · `decomposePar` hierarchical `n (2 2 1)` · **4 ranks** |
| Solver | `simpleFoam` · smoothSolver+GS (U/k/ω) · GAMG+GS (p) |

Hand kernels lived in a thin `libOpenFOAM.so` (interposes Amul / `sumProd` / GS) over baseline `libOpenFOAM-impl.so`. Env gates: `FOAM_RVV_KERNELS`, `FOAM_RVV_AMUL`, `FOAM_RVV_GS`, `FOAM_RVV_PROFILE`.

---

## 1. Auto-vectorize A/B

Same mesh/solve; only `c++OPT` ± `-ftree-vectorize` (full host `-march=…v…`).

| Variant | ExecutionTime | WALL |
| ------- | ------------: | ---: |
| `-ftree-vectorize` | **299.85 s** | 344.1 s |
| `-fno-tree-vectorize` | **300.74 s** | 345.4 s |

**~0%** — auto-vec does not move motorBike.

---

## 2. Hand RVV Amul A/B

CSR-style gather Amul (`vluxei32` + mul + reduce) vs scalar Amul. Profile shows **GS > Amul** on this case.

| Run | WALL | ExecTime | Amul (4 ranks) | GS (4 ranks) |
| --- | ---: | -------: | -------------: | -----------: |
| **RVV Amul on** | **363.8 s** | 319.6 s | **32.5–37.0 s** | 56–65 s |
| **RVV off** | **351.3 s** | 307.1 s | **20.7–25.5 s** | 56–65 s |

Hand RVV Amul is **~50% slower** than scalar → whole solve **~3–4% slower**. GS unchanged.

---

## 3. Hand RVV Gauss–Seidel A/B

Inner face loops only (`nFaces ≥ 4`); Amul forced scalar so the axis is GS alone.

| Run | WALL | ExecTime | GS (4 ranks) |
| --- | ---: | -------: | -----------: |
| **GS RVV on** | **389 s** | 344.9 s | **66.6–75.5 s** |
| **All off** | **387 s** | 342.7 s | **61.9–72.2 s** |

GS mean ~**71 s vs ~67 s** (~5–6% slower); full solve flat / slightly worse.

---

## Why gather RVV loses here

motorBike’s hot linear-algebra is **sparse**:

- **Amul:** irregular `psi[idx[i]]` gathers; short CSR rows.
- **GS:** sequential cell sweep; typically ~4–6 faces per cell — RVV setup + gather/scatter overhead beats a tight scalar face loop on this VLEN.

Useful next levers are **algorithmic** (multicolour / Jacobi-like smoothers, better matrix layout), not more gather microkernels on short rows.

## Reproduce

See [benchmarks/openfoam](https://github.com/opensolvers/benchmarks/tree/main/openfoam) for overlay modules, `FOAM_RVV_*` env, and board run scripts.

**Measured:** 2026-08-01 on Orange Pi RV2.
