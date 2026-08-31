---
title: ESPResSo on SpaceMiT X60
description: ESPResSo 4.2.2 P3M on Orange Pi RV2 — FFTW r5v ~0% end-to-end; local pair-loop optimizations ~1.74× vs EESSI on dense Coulomb.
---

# ESPResSo

[ESPResSo](https://espressomd.org/) is a soft-matter molecular-dynamics package for coarse-grained models — polymers, colloids, and charged fluids — with P3M (Particle–Particle Particle–Mesh) electrostatics.

On the Orange Pi RV2 we ran two lines of work:

1. **FFTW axis** — double-precision `libfftw3.so.3` under one unchanged EESSI `pypresso` binary via `LD_PRELOAD` (same pattern as [GROMACS](gromacs.html) / [FFTW](../scientific-libs/fftw.html)).
2. **Pair-loop axis** — rebuild ESPResSo 4.2.2 with opensolvers patches (slim myconfig, drop `std::function` indirection, monomorph WCA+P3M loop, SR Ewald table) and compare to stock EESSI.

Benchmark source: [opensolvers/benchmarks/espresso](https://github.com/opensolvers/benchmarks/tree/main/espresso). Companion probes: [FFTW](../scientific-libs/fftw.html), [GROMACS](gromacs.html), [ScaFaCoS](../scientific-libs/scafacos.html).

> **Bottom line:** RVV FFTW is **noise (~0%)** on fair pinned P3M runs — same lesson as small-mesh GROMACS and QE. The real win is **source-level pair-loop work**: **~1.74×** vs EESSI on a dense charged fluid (`dense_large`, N≈5528). MPI strong scaling adds **~2.8× at 8 ranks** on the same case.

---

## FFT axis — scalar vs r5v (`run-espresso-fft-ab.sh`)

Swaps only `libfftw3` under EESSI `ESPResSo/4.2.2-foss-2025b`. **Pin mesh/cao/r_cut/alpha** on both sides — auto-tune can pick different meshes if one FFTW is faster during tuning probes.

### Fair pinned results (2026-08-30)

| case | mesh | steps | scalar | r5v | speedup |
| --- | --: | --: | --: | --: | --: |
| N=512, box=20 | 18³ | 1000 | 13.23 s | 13.26 s | **0.998×** |
| N=512, acc=1e−5 | 24³ | 1000 | 34.72 s | 35.56 s | **0.976×** |
| N=4096, box=40 | 32³ | 300 | 57.28 s | 57.41 s | **0.998×** |
| N=4096 | 44³ | 200 | 37.15 s | 36.84 s | **1.008×** |

Energies bit-match. Even at artificial mesh 48³ (Coulomb ~98% of wall), r5v is still **~0.98×**.

`perf` on EESSI `dense_large`: **`libfftw3` ~4%** of samples; top symbols are `add_non_bonded_pair_force`, P3M real-space kernel, and `long_range_kernel`.

---

## Hot path — where the time goes

Multi-model survey (`hotpath_models.py`) on dense charged fluids:

| model | N | Coulomb share of MD |
| --- | --: | --: |
| `lattice512` | 512 | **~88%** |
| `dense_large` | ~5528 | **~86%** |
| dilute samples (`salt_box50`, `electrophoresis`) | — | ~0% (not a Coulomb probe) |

For dense ± fluids, **Coulomb/P3M dominates** — but most of that is Espresso’s pair loop and P3M spread/k-space, not FFTW kernels.

---

## Pair-loop axis — optimized build vs EESSI

Built from ESPResSo 4.2.2 source on RV2 (`build-espresso-opt.sh` → `~/espresso-opt`):

| patch | what it does |
| --- | --- |
| slim myconfig | 7 features (WCA + P3M) vs EESSI’s 34 |
| `ForceKernelRef` | drop `std::function` in Coulomb short-range |
| `add_non_bonded_pair_force_p3m` | monomorph WCA + direct `CoulombP3M::pair_force` |
| SR Ewald table | 4096-bin lookup replaces per-pair `exp` |
| k-space loop | pointer locals in `long_range_kernel` differentiation |

### `dense_large` (N≈5528, mesh 32³, 200 steps, 3 reps)

| build | mean wall | vs EESSI |
| --- | --: | --: |
| EESSI 4.2.2 | 46.8 s | 1.0× |
| opt v1 (ForceKernelRef + slim) | 31.8 s | **1.44×** |
| opt v2 (+ monomorph + SR table) | 28.1 s | **1.66×** |
| opt v3 (+ k-space patch) | **26.8 s** | **1.74×** |

Energies match (`E_tot` / `E_coul` to ~1e−4). `lattice512`: **1.35×** (EESSI ~8.0 s → opt ~6.0 s).

### Levers that did *not* help further

| lever | result |
| --- | --- |
| Lower cao (4/5) + retune | **Slower** — tuner raises `r_cut`; pair work dominates assign savings |
| RVV FFTW on opt build | **~0%** (scalar 27.5 s vs r5v 28.3 s mean) |
| Larger Verlet skin | **Slower** — best skin **0.4** for dense fluid |

After v2 patches, `perf` shows `add_non_bonded_pair_force` and `std::function` _M_invoke gone from the hot list.

---

## MPI strong scaling

Pinned P3M, scalar FFTW, `OMP_NUM_THREADS=1`, OpenMPI on 8× X60:

| np | `dense_large` wall (200 steps) | speedup |
| --: | --: | --: |
| 1 | 46.0 s | 1.00× |
| 2 | 26.2 s | 1.75× |
| 4 | 23.6 s | 1.95× |
| 8 | **16.3 s** | **2.82×** |

Energies match across ranks. For production Coulomb-heavy runs: **MPI + opt binary**, not more FFTW tuning.

---

## Prior note (2026-08-21)

Short auto-tuned run (N=512, 200 steps, mesh 18³) once reported **~1.12×** RVV FFTW; that did not reproduce under longer fair pins with the current xorconj build.

| backend | wall | speedup |
| --- | --: | --: |
| scalar | 2.706 s | — |
| r5v | 2.425 s | **1.12×** |

**Measured:** Orange Pi RV2, EESSI `dev.eessi.io/riscv` 2025.06-001.
