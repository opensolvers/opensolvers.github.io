---
title: MODFLOW 6 on SpaceMiT X60
description: MODFLOW 6 FlexiBLAS A/B on Orange Pi RV2 — USGS ex-gwf-lgrv-lgr via PETSc path; patched RVV ≈ 1.00× (flat).
---

# MODFLOW 6

[MODFLOW 6](https://www.usgs.gov/software/modflow-6-usgs-modular-hydrologic-model) is a groundwater-flow model. On the Orange Pi RV2 the `mf6` binary links PETSc → FlexiBLAS; we A/B OpenBLAS backends under one unchanged install.

Benchmark source: [opensolvers/benchmarks/modflow](https://github.com/opensolvers/benchmarks/tree/main/modflow). Module: `MODFLOW/6.4.4-foss-2023b` on EESSI `riscv.eessi.io` **20240402**.

> **Change one variable.** Same USGS example; only FlexiBLAS OpenBLAS differs.

> **Bottom line:** patched vs scalar ≈ **1.00×** (flat). This workload is not LAPACK-bound on the PETSc/IMS path exercised here. Head-file MD5s match across tags.

---

## Workload

USGS example **`ex-gwf-lgrv-lgr`** (parent 25×90×78 + child 9×61×49) from [modflow6-examples](https://github.com/MODFLOW-ORG/modflow6-examples).

Run with **`mpirun -np 1 mf6 -p`** so the PETSc linear path is active (serial IMS without `-p` finishes the same model in ~54 s). A `.petscrc` requesting MUMPS LU was present but unused by MF6 6.4.4 (IMS shell PC).

---

## Results (2026-08-21)

| Tag | Wall s | MF6 elapsed |
| --- | -----: | ----------- |
| scalar (`RISCV64_GENERIC`) | 189.61 | 3:03.7 |
| stock | 189.00 | 3:03.4 |
| patched RVV | 189.98 | 3:04.3 |

Related: [PETSc](../scientific-libs/petsc.html), [BLAS](../scientific-libs/blas.html).

**Measured:** 2026-08-21 on Orange Pi RV2.
