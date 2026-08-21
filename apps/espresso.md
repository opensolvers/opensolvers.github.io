---
title: ESPResSo on SpaceMiT X60
description: ESPResSo 4.2.2 P3M soft-matter MD on Orange Pi RV2 — double-precision FFTW scalar vs r5v A/B (~1.12×).
---

# ESPResSo

[ESPResSo](https://espressomd.org/) is a soft-matter molecular-dynamics package for coarse-grained models — polymers, colloids, and charged fluids — with P3M (Particle–Particle Particle–Mesh) electrostatics. On the Orange Pi RV2 we A/B **double-precision** `libfftw3.so.3` under one unchanged `pypresso` binary via `LD_PRELOAD`.

Benchmark source: [opensolvers/benchmarks/espresso](https://github.com/opensolvers/benchmarks/tree/main/espresso). Companion FFT probes: [FFTW](../scientific-libs/fftw.html), [GROMACS](gromacs.html), [ScaFaCoS](../scientific-libs/scafacos.html).

> **Change one variable.** Same ESPResSo / OpenBLAS (scalar pinned); only FFTW scalar vs r5v differs. Check `E_coul` / `E_tot` before trusting wall time.

> **Bottom line:** RVV FFTW wins a modest **~1.12×** on the timed MD segment — same ballpark as GROMACS PME 3D-FFT (~1.14–1.23×), diluted by real-space LJ/Coulomb, thermostat, and P3M spread/gather.

---

## Setup

| Piece | Value |
| ----- | ----- |
| Board | Orange Pi RV2 · SpaceMiT X60 |
| ESPResSo | EESSI `ESPResSo/4.2.2-foss-2025b` |
| Workload | charged WCA/LJ fluid · N=512 · box=20 · 200 timed steps · 1 thread |
| Axis | `LD_PRELOAD` scalar vs r5v `libfftw3.so.3` |

---

## Results (2026-08-21)

| Backend | Wall (integrator) | E_coul | E_tot |
| ------- | ----------------: | -----: | ----: |
| scalar `libfftw3` | 2.706 s | −98.404639 | 632.452568 |
| r5v (RVV) `libfftw3` | **2.425 s** | −98.404669 | 632.452480 |
| **speedup** | **1.12×** | Δ ~3e−5 | Δ ~9e−5 |

Energies match across backends.

**Measured:** 2026-08-21 on Orange Pi RV2.
