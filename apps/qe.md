# Quantum ESPRESSO

**Video:** [Stock BLAS MPI_ABORTs a Real DFT on BPI-F3](https://www.youtube.com/watch?v=guf9WCAyYPM) — [all videos](../videos.html)

[Quantum ESPRESSO](https://www.quantum-espresso.org/) is an open-source suite for electronic-structure calculations — plane-wave density-functional theory for materials, surfaces, and molecules. We use `pw.x` SCF as a **whole-application** BLAS backend A/B: swap only OpenBLAS (via FlexiBLAS) under one unchanged binary from the EESSI module (`QuantumESPRESSO/7.5-foss-2025b`).

Benchmark source: [opensolvers/benchmarks/qe](https://github.com/opensolvers/benchmarks/tree/main/qe) — inputs, a Si supercell generator, and `run-qe-ab.sh` / `run-perf-ab.sh` runners.

## Why QE as a probe

[OpenBLAS verification](../scientific-libs/blas.html#verification) isolates BLAS-3; [HPL](hpl.html) and [ELPA](../scientific-libs/elpa.html) probe single HPC solvers; [FFTW](../scientific-libs/fftw.html) covers the FFT half. A full DFT SCF mixes level-3 GEMM (`calbec`, subspace rotation), dense LAPACK diagonalization, latency-bound BLAS-2, MPI, **and a large FFT fraction** — so it shows both whether a buggy vector BLAS breaks a production code, and what fraction of a real run each backend swap actually moves.

## Correctness — stock RVV `gemv_n` breaks a real DFT SCF

`run-qe-ab.sh si-scf.in 4` on SpaceMiT X60 ([Banana Pi F3](../boards/F3.html)), same `pw.x`, backend swapped via FlexiBLAS:

| Backend | Result | Total energy |
| ------- | ------ | ------------ |
| Stock EESSI OpenBLAS 0.3.30, default RVV | `Error in routine inverse_s (1)` → MPI_ABORT | *(none)* |
| Scalar (`RISCV64_GENERIC`) | converged, 7 iterations | **−14.57861334 Ry** |
| Patched RVV (`gemv_n` fix) | converged, 7 iterations (identical trace) | **−14.57861334 Ry** |

The stock vector backend aborts in `inverse_s` (overlap-matrix inversion / Lowdin orthonormalization, which leans on `dgemv`) — the same OpenBLAS 0.3.30 RVV `gemv_n` bug that NaNs [HPL](hpl.html) and [ELPA](../scientific-libs/elpa.html). The patched vector build reproduces the scalar SCF bit-for-bit.

## Performance — 64-atom Si supercell

`run-perf-ab.sh si-super-64.in 4` — gamma-point, np=4, `-ndiag 1` (keeps subspace diagonalization on serial LAPACK→OpenBLAS). Measured on [Banana Pi F3](../boards/F3.html) (SpaceMiT X60, 3.7 GB RAM).

### High band count (`nbnd=272`)

| Routine | Scalar | Patched RVV | Speedup | Kind |
| ------- | -----: | ----------: | ------: | ---- |
| `calbec` (`<β\|ψ>`) | 6.25 s | 3.18 s | **1.97×** | DGEMM |
| Subspace rotation (in `regterg`) | 33.2 s | 16.6 s | **2.0×** | DGEMM |
| `rdiaghg` (dense diag) | 28.2 s | 18.0 s | **1.57×** | LAPACK→BLAS3 |
| `vloc_psi` (apply V to bands) | 52.6 s | 52.4 s | 1.00× | **FFT (untouched)** |
| `fftw` | 54.5 s | 53.8 s | 1.01× | **FFT (untouched)** |
| **`PWSCF` total** | **144.8 s** | **110.3 s** | **1.31×** | whole SCF |

### Default band count (`nbnd=136`)

**67.6 s → 57.0 s = 1.19×** overall (`calbec` 1.90×, `rdiaghg` 1.53×). BLAS routines speed up ~1.5–2.0×; the FFT half does not move with a FlexiBLAS swap. Swapping the FFT library directly ([FFTW](../scientific-libs/fftw.html) `run-qe-fft-ab.sh`) also yields **~0% end-to-end** — QE uses `FFTW_ESTIMATE` on thousands of small transforms, not the `MEASURE` planner where r5v wins **1.06–1.60×** in isolation.

## FFT axis — RVV FFTW also ~0% end-to-end

Serial `pw.x` on [Orange Pi RV2](../boards/RV2.html), BLAS pinned to scalar OpenBLAS, FFT swapped via `LD_PRELOAD` ([`run-qe-fft-ab.sh`](https://github.com/opensolvers/benchmarks/blob/main/fftw/run-qe-fft-ab.sh)):

| Timer | Scalar FFTW | r5v FFTW | Speedup |
| ----- | ----------: | -------: | ------: |
| `fftw` (~45% of run) | 112.24 s | 110.09 s | 1.019× |
| **`PWSCF` (total)** | **248.49 s** | **248.10 s** | **1.002×** |

Energy bit-identical. A microbenchmark FFT win does not automatically become an application win — see also [GROMACS](gromacs.html) for the FFT-axis mirror ( **1.23×** on isolated `PME 3D-FFT`, diluted by scalar `Force`).

## Higher-memory probes (Orange Pi RV2, 7.7 GB)

Same overlay `pw.x`, np=4, `-ndiag 1`, scalar vs patched RVV. These push past what fit on the ~4 GB BPI-F3:

| Probe | atoms / bands / ecut | scalar WALL | patched WALL | **speedup** |
| ----- | -------------------- | ----------: | -----------: | ----------: |
| `si-super-64.in` (baseline) | 64 / 136 / 22 Ry | 70.56 s | 58.89 s | **1.20×** |
| `si-super-64-nbnd272.in` | 64 / **272** / 22 Ry | 186.65 s | 140.84 s | **1.33×** |
| `si-super-64-pbe.in` | 64 / 136 / 22 Ry **PBE** | 121.77 s | 90.11 s | **1.35×** |
| `si-super-64-ecut40-nbnd272.in` | 64 / 272 / **40 Ry** | 411.94 s | 312.74 s | **1.32×** |
| `si-super-216.in` | **216** / 453 / 22 Ry | 1510.31 s | 1033.37 s | **1.46×** |

High-band 64-atom matches the older F3 **1.31×** table (now **1.33×** on RV2). The **216-atom** cell is the clearest whole-app win at **1.46×** — more GEMM-heavy and only feasible with the extra RAM. Best 216-atom breakdown: `calbec` **2.30×**, `rdiaghg` **1.52×**, `fftw` ~400 s unchanged.

Two knobs raise the BLAS fraction: **more bands** (136 → 272: 1.20× → 1.33×) and **bigger supercells** (216-atom → **1.46×**).

## Where QE sits on the BLAS-dilution spectrum

Same X60, patched RVV vs scalar:

| Probe | Speedup | Why |
| ----- | ------: | --- |
| [OpenBLAS verification](../scientific-libs/blas.html#verification) (pure level-3) | ~2.3× | all BLAS-3 |
| [HPL](hpl.html) (Linpack) | ~1.8× | BLAS-3 + `dgemv` panel factorization |
| [ELPA](../scientific-libs/elpa.html) (eigensolver) | ~1.58× | BLAS-3 + BLAS-2 tridiagonalization |
| **QE** (full DFT SCF) | **~1.2–1.5×** | BLAS + ~40–50% FFT + MPI |

Each step down adds more non-BLAS / latency-bound work, diluting the BLAS-3 peak. FFT drop-in swaps still ~0% end-to-end under `FFTW_ESTIMATE`; BLAS wins grow when the cell is large enough that GEMM owns more of the wall.

## Reproducing

```bash
module load QuantumESPRESSO/7.5-foss-2025b
curl -O https://pseudopotentials.quantum-espresso.org/upf_files/Si.pz-vbc.UPF

# Correctness (np=4)
RVV_LIB=/path/to/patched/libopenblas.so ./run-qe-ab.sh si-scf.in 4

# Performance (np=4)
RVV_LIB=/path/to/patched/libopenblas.so ./run-perf-ab.sh si-super-64.in 4
```

**Toolchain:** QuantumESPRESSO 7.5 / foss-2025b (EESSI), FlexiBLAS 3.4.5, OpenBLAS 0.3.30. Patched vector backend = OpenBLAS 0.3.30 with the RISC-V `gemv_n` NaN fix backported — same build used by [OpenBLAS verification](../scientific-libs/blas.html#verification), [HPL](hpl.html), and [ELPA](../scientific-libs/elpa.html).
