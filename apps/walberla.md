---
title: waLBerla on SpaceMiT X60
description: waLBerla 7.2 RVV / auto-vec campaign on Orange Pi RV2 — BasicLBM ISA tag, HeatEquation 1.64×, UniformGrid collide 1.54×, SoA auto-vec vs hand simd.
---

# waLBerla

[waLBerla](https://www.walberla.net/) (widely applicable Lattice Boltzmann from Erlangen) is a C++ framework for lattice Boltzmann methods and structured-grid PDEs — fluids, multiphase flow, and related multiphysics. We measure **7.2** tutorials on the Orange Pi RV2 (SpaceMiT K1 / X60, RVV 1.0, VLEN=256), under EESSI `dev.eessi.io/riscv` (`2025.06-001`, `foss-2025b`).

Benchmark source: [opensolvers/benchmarks/walberla](https://github.com/opensolvers/benchmarks/tree/walberla-rvv-autovec-results/walberla).

> **Change one variable.** Same source / OpenMPI / prm — only the **ISA tag / `-march`** differs: stock EESSI (`rv64gc`, no `v`) vs local rebuild (`-march=rv64gcv`, ELF tag includes `v1p0` + `zve*` / `zvl*`).

> **Bottom line:** BasicLBM ISA-tag alone is only ~**1–4%**. Contiguous auto-vec wins show up on HeatEquation (**1.64×** np1) and UniformGrid `--not-fused` (**1.30×** WALL / **1.54×** collide). Hand `simd::double4_t` RVV loses to FORCE_SCALAR; plain SoA auto-vec is ~**2.4×** vs novec (~**9×** vs hand simd). Splitting collide/stream for vectorizable collide costs more than it gains vs fused stock on bandwidth-bound D2Q9.

Contrast sparse gather regressions in [OpenFOAM](openfoam.html).

---

## BasicLBM ISA A/B (`01_BasicLBM`)

| Variant | np | WALL (s) | vs stock |
| ------- | -: | -------: | -------: |
| stock-gc | 1 | **129** | — |
| gcv | 1 | **127** | **~1.6%** (~1.02×) |
| stock-gc | 4 | **47** | — |
| gcv | 4 | **45** | **~4.3%** (~1.04×) |

Domain: 320 000 cells · 2000 timesteps. MPI np=4 vs np=1 is ~**2.7–2.9×** on this small 2-D domain. Stock is already Release/`-O2`; the axis is mostly **ISA enablement**, not a hand RVV kernel — and classic D2Q9 is **memory-bandwidth heavy**.

---

## HeatEquation Jacobi (`02_HeatEquation`)

Stock D2Q5 Jacobi tutorial (VTK off). Domain **200×200**, 10 steps × 2000 Jacobi. Checksum identical across variants.

| Variant | np | WALL (s) | vs gc |
| ------- | -: | -------: | ----: |
| gc (`rv64gc`) | 1 | **35.57** | — |
| gcv (`rv64gcv`) | 1 | **21.76** | **1.64×** |
| novec (`rv64gcv -fno-tree-vectorize`) | 1 | **37.61** | 0.95× |
| gc | 4 | **17.55** | — |
| gcv | 4 | **16.65** | **1.05×** |

Jacobi `operator()` contains RVV under gcv (`vle64`/`vse64`/`vfdiv`); none under gc / true novec.

---

## UniformGrid `--not-fused` (`UniformGridBenchmark`)

Official waLBerla 7.2 benchmark; `--not-fused` separates SoA `SplitPureSweep` collide from stream. SRT D3Q19 · **64³**/block.

| Mode | Variant | np | WALL (s) | collide (s) | ≈ MLUPS | vs gc |
| ---- | ------- | -: | -------: | ----------: | ------: | ----- |
| **not-fused** | gc | 1 | **30** | **4.01** | 1.81 | — |
| **not-fused** | gcv | 1 | **23** | **2.61** | 2.32 | **1.30×** WALL · **1.54×** collide |
| fused | gc | 1 | 25 | — | 2.25 | — |
| fused | gcv | 1 | 21 | — | 2.70 | 1.19× WALL |
| **not-fused** | gcv | 4 | 26 | 9.76 | 5.33 | 1.04× WALL · 1.24× collide |

Collide auto-vec under gcv confirmed (`SplitPureSweep.impl.h` VL vectors; objdump `vle64=1822` vs gc `0`). Stream stays ~flat (gather-bound). Narrative: between HeatEquation (~1.64×) and BasicLBM (~flat).

---

## SoA microbenches — prefer plain auto-vec over hand `simd::`

Standalone D2Q9-ish collide TU, contiguous PDF arrays, GCC 14.3 `-march=rv64gcv`:

| Variant | BEST WALL (s) | ≈ MLUPS |
| ------- | ------------: | ------: |
| **AUTOVEC** (`-ftree-vectorize`, plain `double*`) | **1.58** | **13.3** |
| NOVEC (`-fno-tree-vectorize`) | **3.76** | 5.57 |
| Hand `simd::double4_t` (`vector_size(32)`) | 14.60 | 1.44 |
| FORCE_SCALAR (+ auto-vec via Scalar.h) | 8.34 | 2.52 |

**AUTOVEC ≈ 2.4×** over NOVEC and **≈ 9×** over hand `double4_t`. Recommendation: for GCC/RVV SoA collide work, prefer contiguous `double*` + auto-vec over chasing `walberla::simd::double4_t`.

### Split collide/stream vs fused CellwiseSweep

Unlocking a vectorizable collide by splitting stream then collide **loses** whole-app: stock fused **133.8 s** vs split **175.9 s** (np1) — extra full-field stream pass costs more than the collide auto-vec wins on bandwidth-bound D2Q9.

---

## Reading

This is the **contiguous-stencil** counterpart to OpenFOAM’s sparse Amul/GS story: dense cell arrays can benefit from `-march=rv64gcv` + auto-vec when the hot loop is contiguous; gather/stream and fused bandwidth-bound LBM dilute or erase that win. A larger RVV story needs explicit vector kernels (or codegen that emits them), plus bandwidth-aware sizing — not just an ISA tag on BasicLBM.

## Reproduce

See [benchmarks/walberla](https://github.com/opensolvers/benchmarks/tree/walberla-rvv-autovec-results/walberla) for prm files, build scripts, and `results/`.

**Measured:** 2026-08-02 on Orange Pi RV2.
