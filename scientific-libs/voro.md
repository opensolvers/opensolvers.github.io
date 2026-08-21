---
title: Voro++ on SpaceMiT X60
description: Voro++ 0.4.6 RVV auto-vec A/B on Orange Pi RV2 — GCC emits RVV in cell.o but ~0.99× vs novec (negative control).
---

# Voro++

[Voro++](http://math.lbl.gov/voro++/) is a C++ library for cell-based 3D Voronoi tessellation. Stock EESSI `Voro++/0.4.6-GCCcore-14.3.0` is built `-march=rv64gc` only; we rebuild from upstream for a compiler-flag A/B on the Orange Pi RV2.

Benchmark source: [opensolvers/benchmarks/voro++](https://github.com/opensolvers/benchmarks/tree/main/voro%2B%2B).

> **Change one variable.** Same source / problem size; only `-march` / `-ftree-vectorize` differs.

> **Bottom line:** GCC **does** emit RVV in `cell.o` (309 RVV-ish insns), but wall time is **~0.99×** vs novec — flat / slightly slower. Useful **negative control** next to [waLBerla](../apps/walberla.html) SoA auto-vec wins.

---

## Results (2026-08-21)

N=20 000 cells, best of several reps. Checksums match (`VVOL=1`, `FACES=297872`).

| Variant | Flags | `cell.o` RVV-ish | best ms |
| ------- | ----- | ---------------: | ------: |
| **novec** | `-O3 -march=rv64gc -fno-tree-vectorize` | 0 | **1244.8** |
| **gcv** | `-O3 -march=rv64gcv -ftree-vectorize` | 309 | 1261.6 (**0.99×**) |

Interpretation: Voro++ is irregular — per-cell plane cuts, short variable-length loops, pointer-heavy. Auto-vec emits instructions without paying off.

Contrast contiguous SoA wins in [waLBerla](../apps/walberla.html) and sparse regressions in [OpenFOAM](../apps/openfoam.html).

**Measured:** 2026-08-21 on Orange Pi RV2.
