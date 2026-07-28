---
title: Kokkos on SpaceMiT X60
description: Kokkos as used by LAMMPS on Orange Pi RV2 — OpenMP/Serial execution spaces, no RVV SIMD backend, Pair-dominated profile, and hand RVV LJ/EAM Pair results.
---

# Kokkos

[Kokkos](https://kokkos.org/) portable parallel dispatch as used by [LAMMPS](../apps/lammps.html) on the Orange Pi RV2 (SpaceMiT X60, RVV 1.0, VLEN=256) under the EESSI RISC-V stack.

Benchmark / notes source: [opensolvers/benchmarks/kokkos](https://github.com/opensolvers/benchmarks/tree/main/kokkos) (`README.md`, `RESULTS.md`). Hand RVV Pair plugins (not Kokkos SIMD): [`lammps/rvv-lj/`](https://github.com/opensolvers/benchmarks/tree/main/lammps/rvv-lj), [`lammps/rvv-eam/`](https://github.com/opensolvers/benchmarks/tree/main/lammps/rvv-eam).

Whole-app MD scaling (serial / Kokkos8 / MPI8): [LAMMPS](../apps/lammps.html). Analogous “hand SIMD the hot kernel” success: [GROMACS Force](../apps/gromacs.html).

---

## What Kokkos does in LAMMPS

| Primitive | Role |
| --------- | ---- |
| `parallel_for` | Pair / Neigh / Integrate loops over atoms |
| `parallel_reduce` | Energies, virials, reductions |
| `parallel_scan` | Rare on the Pair path |

**Execution spaces on our build:** OpenMP + Serial only.  
**Arch:** `Kokkos_ARCH_EASYBUILD_GENERIC` — avoids `-mcpu=native` (RISC-V GCC 14.3 rejects it); march comes from EESSI flags (`…cv…` / `zve*`).

**SIMD layer (Kokkos 4.6.2 bundled with LAMMPS):** backends are **AVX2 / AVX512 / NEON / Scalar** only — **no RVV abi**. LAMMPS `PairComputeFunctor` does **not** use `Kokkos::Experimental::simd`; it runs scalar FP inside `parallel_for`. Any RVV in `liblammps.so` today is **GCC auto-vec** (`-ftree-vectorize` + `-march=…cv…`), not a deliberate Kokkos SIMD path.

Stock Kokkos on this board is a **portable OpenMP vehicle**, not an RVV vectorizer — the same lesson as GROMACS before `impl_riscv_rvv`.

---

## Hot path — Pair dominates

Board profiling (`in.lj` / `in.melt`, Kokkos OpenMP):

| Activity | Share | Notes |
| -------- | ----: | ----- |
| **Pair** (`PairLJCutKokkos` / `PairComputeFunctor`) | **~84–87%** | Dominant |
| **Neigh** | ~8–12% | List build / distance |
| Comm / Modify / Other | small | |

Inner loop (`pair_kokkos.h`): neighbor gather → cutoff → `compute_fpair` (LJ) → accumulate `f[i]` (Newton-on also scatters to `f[j]`).

### Why stock Kokkos Pair is a weak RVV target *as written*

- Gather/scatter of positions/forces (irregular neighbor indices)
- Variable `jnum`, type-dependent params, `special_lj` → divergence
- Newton-on **atomics** on `f[j]` fight wide SIMD
- No Kokkos RVV SIMD backend to express a vector Pair cleanly

Auto-vec alone will not deliver OpenBLAS-like gains. The workable path is **hand layout + hand RVV** on the Pair math.

---

## Hand RVV Pair — results so far

### LJ/cut — [`lammps/rvv-lj`](https://github.com/opensolvers/benchmarks/tree/main/lammps/rvv-lj)

| Build | Scope | Result |
| ----- | ----- | ------ |
| EESSI GCC 14.3 microbench | force-on-i vs naive scalar | **~1.61–1.64×** |
| `lj/cut/rvv` plugin vs stock `lj/cut` | in-LAMMPS Pair, 4000 atoms | **~1.02×** (near parity) |

Microbench (force-on-i, SoA tiles, `taskset -c 0`, EESSI GCC 14.3):

| n | nnz | scalar ns/pair | RVV ns/pair | Speedup | max‖Δf‖ |
| -: | --: | -------------: | ----------: | ------: | ------: |
| 2048 | 98426 | 58.7 | 35.8 | **1.64×** | 6.75×10⁻¹⁴ |
| 4096 | 196644 | 58.9 | 36.4 | **1.62×** | 4.10×10⁻¹⁴ |

In-app LJ is gather/scatter limited; stock is already `-march=…cv…` auto-vec’d.

### EAM — [`lammps/rvv-eam`](https://github.com/opensolvers/benchmarks/tree/main/lammps/rvv-eam)

Cu `Cu_u3.eam`, 864 atoms, 100 steps, 1 core, force-only; forces **bit-exact** vs stock:

| Style | Pair time | vs `eam` |
| ----- | --------: | -------: |
| `eam` | 0.780 s | 1.00× |
| **`eam/rvv`** | 0.614 s | **1.27×** |
| `eam/opt` | 0.574 s | 1.36× |

EAM is ~**96%** Pair — a better RVV target than LJ. Hand RVV beats stock `eam` but still trails the OPT package (~7%).

---

## Board LAMMPS / Kokkos snapshot

| Item | Status |
| ---- | ------ |
| Version | 22 Jul 2025 Update 4 + Kokkos 4.6.2 |
| Accelerator (`lmp -h`) | `KOKKOS package API: OpenMP Serial`; FFT = FFTW3 |
| CVMFS | No `LAMMPS` in `dev.eessi.io/riscv` easystack yet (local overlay build) |
| `in.melt` | **PASSED** |

## EESSI on RISC-V

```bash
export EESSI_VERSION_OVERRIDE=2025.06-001
source /cvmfs/software.eessi.io/versions/2025.06/init/lmod/bash
module load GCC/14.3.0          # or foss/2025b for full LAMMPS
```

Prefer EESSI **GCC 14.3** over board gcc 13.3; never pipe `source`/`module` into `tail`.

## Backlog

1. Close `eam/rvv` vs `eam/opt` (~7%)
2. Upstream Kokkos **`SIMD_RVV`** backend — unlocks SIMD-aware kernels later
3. Neigh distance filter — lower ROI than Pair
4. Further LJ tuning — low ROI after ~1.02× in-app

**Toolchain:** EESSI `2025.06-001` / GCC 14.3.0, Kokkos 4.6.2, Orange Pi RV2 (SpaceMiT X60).
