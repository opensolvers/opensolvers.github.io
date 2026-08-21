---
title: PLUMED on SpaceMiT X60
description: PLUMED 2.9.4 FlexiBLAS A/B on Orange Pi RV2 — SPRINT / CONTACT_MATRIX CV path; patched RVV OpenBLAS 1.25× vs scalar.
---

# PLUMED

[PLUMED](https://www.plumed.org/) is an enhanced-sampling and collective-variable library used with MD engines (GROMACS, LAMMPS, …). On the Orange Pi RV2 we swap the OpenBLAS backend under an unchanged `plumed driver` via FlexiBLAS.

Benchmark source: [opensolvers/benchmarks/plumed](https://github.com/opensolvers/benchmarks/tree/main/plumed). Module: `PLUMED/2.9.4-foss-2025b` on EESSI `2025.06-001`.

> **Change one variable.** Same PLUMED binary / trajectory; only FlexiBLAS OpenBLAS (scalar vs patched RVV) differs.

> **Bottom line:** per-frame **SPRINT** on a `CONTACT_MATRIX` (dense N×N LAPACK) — patched RVV **1.25×** vs scalar. Checksums match.

---

## Workload

N particles, synthetic cubic LJ liquid. Largest-eigenvalue topological CVs → dense LAPACK through FlexiBLAS:

```
d: DENSITY SPECIES=1-N
mat: CONTACT_MATRIX ATOMS=d SWITCH={RATIONAL R_0=1.2 D_MAX=2.0}
ss: SPRINT MATRIX=mat
```

Note: bare `CONTACT_MATRIX ATOMS=1-N` + `SPRINT` segfaults on this build; the `DENSITY` wrapper is required.

---

## Results (2026-08-21)

| Backend | Wall (N=200, 20 frames) |
| ------- | ----------------------: |
| scalar (`RISCV64_GENERIC`) | 80.36 s |
| patched RVV | **64.11 s** (**1.25×**) |

Checksum: `sum(ss.*)=975.389111` on last frame (both backends).

Related BLAS context: [BLAS](blas.html), [PETSc](petsc.html).

**Measured:** 2026-08-21 on Orange Pi RV2.
