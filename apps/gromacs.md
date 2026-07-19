# GROMACS

[GROMACS](https://www.gromacs.org/) `mdrun` PME molecular dynamics — a **whole-application FFT backend A/B**. Swaps only the single-precision FFTW library (`libfftw3f.so.3`) under one unchanged binary via `LD_PRELOAD`; GROMACS comes from the EESSI module (`GROMACS/2026.2-foss-2025b`).

Benchmark source: [opensolvers/benchmarks/gromacs](https://github.com/opensolvers/benchmarks/tree/main/gromacs) — `gen-water-box.sh` and `run-gmx-fft-ab.sh`.

This is the **FFT-axis** companion to [Quantum ESPRESSO](qe.html)'s BLAS-axis probe. FFT microbench numbers: [FFTW](../scientific-libs/fftw.html).

## Why GROMACS as an FFT probe

PME splits long-range electrostatics into a real-space **`Force`** term and a reciprocal-space **mesh** part whose cost is a pair of 3D FFTs per step. On this build GROMACS reports **`SIMD instructions: None`** — force kernels are scalar — so swapping `libfftw3f` moves only the `PME 3D-FFT` / `PME mesh` rows in GROMACS's cycle table; `Force`, neighbor search, and constraints are built-in controls.

## Correctness — RVV FFT reproduces scalar MD

Same `mdrun`, same `md.tpr`, `libfftw3f` swapped (17.2k-atom SPC water, 2000 steps):

| FFT backend | Avg potential energy | Δ vs scalar |
| ----------- | -------------------: | ----------: |
| Scalar (`libfftw3f`, ~944 KB) | **−236755 kJ/mol** | — |
| r5v / RVV (`libfftw3f`, ~14 MB, 282k RVV instr) | **−236592 kJ/mol** | **0.069%** |

Physically the same trajectory — expected single-precision rounding between codelet sets.

## Performance — SPC water PME (Orange Pi RV2)

Serial, 48×48×48 PME grid. BLAS pinned to scalar OpenBLAS via FlexiBLAS; only `LD_PRELOAD` swaps FFT:

| Activity | Scalar | r5v (RVV) | Speedup | Kind |
| -------- | -----: | --------: | ------: | ---- |
| **`PME 3D-FFT`** | 21.095 s | 17.152 s | **1.23×** | **FFT (swapped axis)** |
| `PME mesh` (total) | 73.281 s | 69.792 s | 1.05× | FFT + spread/gather/solve |
| `PME solve Elec` | 9.116 s | 9.139 s | 1.00× | scalar (control) |
| **`Force`** | 821.35 s | 819.97 s | 1.00× | **scalar kernels (90% of run)** |
| `Neighbor search` | 6.731 s | 6.714 s | 1.00× | scalar (control) |

The **`Force` row confirms the A/B** — 821.35 vs 819.97 s (0.17%, noise). RVV wins **~1.23×** on the isolated 3D-FFT step, but `Force` dominates the whole run.

## Where GROMACS sits on the dilution spectrum

| Probe | Axis | Isolated speedup | Whole-app effect |
| ----- | ---- | ---------------: | ---------------- |
| [OpenBLAS verification](../scientific-libs/blas.html#verification) | BLAS | ~2.3× | (pure kernel) |
| [QE](qe.html) (DFT SCF) | BLAS | ~1.5–2.0× on BLAS routines | ~1.2–1.3× (FFT half untouched) |
| [QE + FFTW](../scientific-libs/fftw.html) | FFT | ~1.02× on `fftw` | **~0%** (`FFTW_ESTIMATE`) |
| **GROMACS** (this page) | **FFT** | **~1.23× on `PME 3D-FFT`** | small (`Force` = 90%) |

GROMACS is the mirror of QE: QE speeds up BLAS and is diluted by FFT; GROMACS speeds up FFT and is diluted by scalar force kernels.

## Gotchas

- GROMACS mixed precision links **`libfftw3f.so.3`** (single precision), not the double-precision `libfftw3.so.3` from the QE/FFTW work.
- Fixed `-nsteps` keeps FFT call count identical between backends (4002 here).
- RVV `libfftw3f` build is slow to compile (~2 h on one X60); scalar ~15 min.

## Reproduce

```bash
module load GROMACS/2026.2-foss-2025b
./gen-water-box.sh
bash run-gmx-fft-ab.sh v1
```

**Toolchain:** GROMACS 2026.2 / foss-2025b (EESSI), FFTW 3.3.10 `libfftw3f`, Orange Pi RV2 (SpaceMiT X60).
