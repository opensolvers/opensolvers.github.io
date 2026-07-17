# Quantum ESPRESSO

[Quantum ESPRESSO](https://www.quantum-espresso.org/) `pw.x` plane-wave DFT SCF — a **whole-application** BLAS backend A/B. Swaps only the OpenBLAS implementation (via FlexiBLAS) under one unchanged `pw.x` binary; QE comes from the EESSI module (`QuantumESPRESSO/7.5-foss-2025b`).

Benchmark source: [opensolvers/benchmarks/qe](https://github.com/opensolvers/benchmarks/tree/main/qe) — inputs, a Si supercell generator, and `run-qe-ab.sh` / `run-perf-ab.sh` runners.

## Why QE as a probe

[`dgemm`](../scientific-libs/dgemm.html) isolates BLAS-3; [HPL](hpl.html) and [ELPA](../scientific-libs/elpa.html) probe single HPC solvers; [FFTW](../scientific-libs/fftw.html) covers the FFT half. A full DFT SCF mixes level-3 GEMM (`calbec`, subspace rotation), dense LAPACK diagonalization, latency-bound BLAS-2, MPI, **and a large FFT fraction** — so it shows both whether a buggy vector BLAS breaks a production code, and what fraction of a real run each backend swap actually moves.

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

**67.6 s → 57.0 s = 1.19×** overall (`calbec` 1.90×, `rdiaghg` 1.53×). BLAS routines speed up ~1.5–2.0×; the FFT half of the run does not move with a FlexiBLAS swap — see [FFTW](../scientific-libs/fftw.html) for RVV FFT tuning on the same hardware (**1.06–1.60×** under `FFTW_MEASURE`, plus **3–5×** from planner choice).

## Where QE sits on the BLAS-dilution spectrum

Same X60, patched RVV vs scalar:

| Probe | Speedup | Why |
| ----- | ------: | --- |
| [DGEMM](../scientific-libs/dgemm.html) (pure level-3) | ~2.3× | all BLAS-3 |
| [HPL](hpl.html) (Linpack) | ~1.8× | BLAS-3 + `dgemv` panel factorization |
| [ELPA](../scientific-libs/elpa.html) (eigensolver) | ~1.58× | BLAS-3 + BLAS-2 tridiagonalization |
| **QE** (full DFT SCF) | **~1.2–1.3×** | BLAS + ~40–50% FFT + MPI |

Each step down adds more non-BLAS / latency-bound work, diluting the BLAS-3 peak. QE is the most realistic whole-application figure — and the reason a raw `dgemm` A/B *overstates* what a real code sees.

## Reproducing

```bash
module load QuantumESPRESSO/7.5-foss-2025b
curl -O https://pseudopotentials.quantum-espresso.org/upf_files/Si.pz-vbc.UPF

# Correctness (np=4)
RVV_LIB=/path/to/patched/libopenblas.so ./run-qe-ab.sh si-scf.in 4

# Performance (np=4)
RVV_LIB=/path/to/patched/libopenblas.so ./run-perf-ab.sh si-super-64.in 4
```

**Toolchain:** QuantumESPRESSO 7.5 / foss-2025b (EESSI), FlexiBLAS 3.4.5, OpenBLAS 0.3.30. Patched vector backend = OpenBLAS 0.3.30 with the RISC-V `gemv_n` NaN fix backported — same build used by [DGEMM](../scientific-libs/dgemm.html), [HPL](hpl.html), and [ELPA](../scientific-libs/elpa.html).
