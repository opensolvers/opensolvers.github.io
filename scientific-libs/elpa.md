# ELPA

[ELPA](https://elpa.mpcdf.mpg.de/) is a dense real-symmetric eigensolver used in DFT codes (CP2K, VASP, ELSI). It exercises a **mix of BLAS-2 and BLAS-3** (`dsymv`, `dsyr2k`, `dgemm`, `dtrmm`) — a more representative BLAS probe than raw `dgemm`.

Benchmark source: [opensolvers/benchmarks/elpa](https://github.com/opensolvers/benchmarks/tree/main/elpa).

## Orange Pi RV2 (SpaceMiT X60, 8 cores)

`na=3000`, 1-stage solver, 8 threads, single MPI rank. Backends swapped via FlexiBLAS — no rebuild.

| Backend | Time | Correctness |
| ------- | ---- | ----------- |
| Stock EESSI OpenBLAS 0.3.30, default RVV | 42.17 s | **`ev0=nan`, finite=0 — FAIL** |
| Scalar (`RISCV64_GENERIC`) | 54.92 s | finite=1 |
| Patched RVV (`gemv_n` fix) | **34.81 s** | finite=1 |

The stock vector backend is faster than scalar but **wrong** — the same OpenBLAS 0.3.30 `gemv_n` NaN bug that breaks [HPL](../apps/hpl.html). With the patch, RVV is **1.58×** faster than scalar (54.92 / 34.81) and numerically correct.

Speedup is smaller than pure `dgemm` (~2.3×) because tridiagonalization is latency-bound BLAS-2 — by design.

See also the [BLAS overview](blas.html) and [LAPACK](lapack.html) NumPy probe.
