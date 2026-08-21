---
title: ScaFaCoS on SpaceMiT X60
description: ScaFaCoS 1.0.4 P3M Coulomb solver on Orange Pi RV2 — FFTW scalar vs r5v A/B (~0.99× serial; near-field dilutes FFT).
---

# ScaFaCoS

[ScaFaCoS](http://www.scafacos.de/) is a library of scalable Fast Coulomb Solvers. On the Orange Pi RV2 we time the **P3M** method and A/B double-precision `libfftw3.so.3` under one unchanged binary via `LD_PRELOAD`.

Benchmark source: [opensolvers/benchmarks/scafacos](https://github.com/opensolvers/benchmarks/tree/main/scafacos). Module: `ScaFaCoS/1.0.4-foss-2025b` (library-only on EESSI). Companion FFT probes: [ESPResSo](../apps/espresso.html), [GROMACS](../apps/gromacs.html), [FFTW](fftw.html).

> **Bottom line:** serial P3M (np=1) — r5v **~0.99×** vs scalar; energies match. Same lesson as small-mesh GROMACS/ESPResSo: near-field + bookkeeping dilute the FFT. MPI np=4 row is not a pure FFT A/B (`libfftw3_mpi` stays stock CVMFS).

---

## Results (2026-08-21)

### np=1 (cleanest `LD_PRELOAD`)

N=32³ = 32768 particles, 10 timed runs.

| Backend | per-run wall | E (½ Σ q·φ) |
| ------- | -----------: | ----------: |
| scalar `libfftw3` | **5.248 s** | −916228.021 |
| r5v (RVV) | 5.290 s (**0.99×**) | −916228.024 |

### np=4 (mixed)

N=24³ = 13824; `libfftw3_mpi` from EESSI (no r5v MPI build on board).

| Backend | per-run wall | E |
| ------- | -----------: | -: |
| scalar preload | **0.738 s** | −289900.273 |
| r5v preload | 0.820 s (**0.90×**) | −289900.273 |

Build note: CMake pulls FMM → GlobalArrays/`armci`; board module has no GA — thin `armci_stubs.c` satisfies the linker for the **P3M** path only (`method=fmm` unsupported with stubs).

**Measured:** 2026-08-21 on Orange Pi RV2.
