---
title: LAMMPS on SpaceMiT X60
description: RVV-Kokkos LAMMPS whole-app MD on Orange Pi RV2 — five upstream bench workloads, serial vs Kokkos-OpenMP vs MPI parallel scaling.
---

# LAMMPS

[LAMMPS](https://www.lammps.org/) molecular dynamics on the SpaceMiT X60 — a **whole-application throughput** probe with an **RVV-vectorized Kokkos** build, not a drop-in library A/B like [GROMACS](gromacs.html) FFT or [QE](qe.html) BLAS.

Benchmark source: [opensolvers/benchmarks/lammps](https://github.com/opensolvers/benchmarks/tree/main/lammps) — `run-lammps-bench.sh` runs the five upstream `bench/` inputs × three modes (serial / Kokkos-OpenMP / MPI).

The binary is genuinely vectorized (`Tag_RISCV_arch` includes `v1p0` / `zve64d`; ~53k `vsetvli`, ~12k `vf*` FMAs in `liblammps.so.0`). Speedups below are **parallel scaling** (1→8 cores) on that vectorized build — **not** a vector-vs-scalar A/B.

---

## Five upstream benches

Same default problem size everywhere: **32000 atoms, 100 steps**.

| Bench | Potential / physics | Kernel character |
| ----- | ------------------- | ---------------- |
| `lj` | Lennard-Jones (Melt) | short-range pair — canonical MD baseline |
| `eam` | embedded-atom metal (Cu) | many-body metal — heavier per-pair |
| `chain` | bead-spring polymer (FENE) | bonded + short-range — communication-bound |
| `chute` | granular chute flow | granular contact, gravity-driven |
| `rhodo` | rhodopsin (CHARMM + PPPM) | full biomolecular: bonded + long-range Coulomb |

---

## Performance — Orange Pi RV2 (8× X60 @ ~1.6 GHz)

LAMMPS `Loop time` and `Performance:` (katom-step/s). Speedup vs that bench’s serial run.

| Benchmark | Mode | Loop (s) | katom-step/s | vs serial |
| --------- | ---- | -------: | -----------: | --------: |
| **lj** | serial | 15.48 | 206.7 | 1.00× |
| | kokkos8 | 2.496 | 1282 | **6.20×** |
| | mpi8 | 3.003 | 1065 | 5.15× |
| **eam** | serial | 41.69 | 76.8 | 1.00× |
| | kokkos8 | 5.784 | 553.3 | **7.21×** |
| | mpi8 | 7.559 | 423.4 | 5.52× |
| **chain** | serial | 8.950 | 357.5 | 1.00× |
| | kokkos8 | 2.213 | 1446 | 4.04× |
| | mpi8 | 1.578 | 2028 | **5.67×** |
| **chute** | serial | 6.491 | 493.0 | 1.00× |
| | kokkos8 | 1.547 | 2069 | 4.20× |
| | mpi8 | 1.325 | 2414 | **4.90×** |
| **rhodo** | serial | 321.79 | 9.9 | 1.00× |
| | kokkos8 | 59.10 | 54.1 | 5.44× |
| | mpi8 | 54.17 | 59.1 | **5.94×** |

Kokkos = `-k on t 8 -sf kk` / `OMP_NUM_THREADS=8`. MPI = `mpirun -np 8`.

### Two regimes on 8 cores

- **Compute-bound pair potentials (`lj`, `eam`)** favour **Kokkos/OpenMP** — RVV force kernels + shared-memory threads hit **6.2× / 7.2×**, beating MPI. `eam` is the best threaded scaler (**7.21×**).
- **Communication / bonded-heavy (`chain`, `chute`, `rhodo`)** favour **MPI** — domain decomposition wins when the kernel is lighter and neighbor/ghost exchange dominates (`chain` **5.67×**, `rhodo` **5.94×**).

`rhodo` is the absolute-cost outlier (~20× slower than `lj` serial for the same atom count) — full CHARMM + PPPM — but still reaches a usable ~54 s under either parallel back-end.

---

## Where the RVV work lives

On single-thread Kokkos `lj` (`-k on t 1 -sf kk`), LAMMPS timers and gdb stack sampling agree:

| Phase (LAMMPS timer) | % of wall | Kernel |
| -------------------- | --------: | ------ |
| **Pair** | **84.4%** | `PairLJCutKokkos` — autovectorized lj/cut force |
| Neigh | 11.8% | neighbor-list build |
| Comm / Modify / other | ~3.8% | ghosts, NVE, output |

gdb leaf samples put `PairComputeFunctor<PairLJCutKokkos>::operator()` first (~46%, plus stripped leaves on the same path). Kokkos is header-only and inherits `-march=…v`, so the ELF’s `vf*` FMAs run **inside this pair functor** — not in FFTW/OpenBLAS (`lj` has no FFT/BLAS symbols; FFTW only matters for `rhodo` PPPM).

---

## Building on RISC-V

No upstream LAMMPS module in `dev.eessi.io/riscv` yet. This binary used a **custom easyconfig** (LAMMPS 22Jul2025 update4, Kokkos 4.6.2, GCC 14.3, OpenMPI 5.0.8) against `foss-2025b`. Five RISC-V-specific fixes were required — details in the [benchmarks README](https://github.com/opensolvers/benchmarks/blob/main/lammps/README.md#building-on-risc-v-custom-easyconfig-foss-2025b) (`kokkos_arch=EASYBUILD_GENERIC`, CVMFS `find_*` storm, pin binutils, `USE_SPGLIB=OFF`, drop `MDI`).

## Gotchas

- Speedups are **parallel scaling on an RVV build**, not RVV-vs-scalar uplift.
- `ctest` blocks EasyBuild install (3/571 fail for non-compute reasons) — use `--skip-test-step` or install the built tree manually.
- Detached shells need a login shell for modules; put `set -u` **after** sourcing EESSI/Lmod.
- Prefer `$EBROOTOPENMPI/bin/mpirun` over bare `mpirun` on `PATH`.
- Stage `data.{chain,chute,rhodo}` and `Cu_u3.eam` next to the `in.*` inputs.

## Reproduce

```bash
# once upstreamed:
module load LAMMPS/22Jul2025_update4-foss-2025b-kokkos
# until then, point LMP / MPIRUN at a manual prefix

cp <lammps-src>/bench/in.{lj,eam,chain,chute,rhodo} .
cp <lammps-src>/bench/Cu_u3.eam <lammps-src>/bench/data.{chain,chute,rhodo} .
./run-lammps-bench.sh
```

**Toolchain:** LAMMPS 22Jul2025 update4 / foss-2025b (custom easyconfig on EESSI `dev.eessi.io/riscv`), Kokkos 4.6.2, GCC 14.3.0, OpenMPI 5.0.8, Orange Pi RV2 (SpaceMiT X60).
