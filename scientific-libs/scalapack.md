# ScaLAPACK

[ScaLAPACK](https://www.netlib.org/scalapack/) `PDSYEV` — a distributed dense real-symmetric eigenproblem on a 2D block-cyclic process grid. A **pure-MPI** proxy for node-local BLAS performance and correctness.

Benchmark source: [opensolvers/benchmarks/scalapack](https://github.com/opensolvers/benchmarks/tree/main/scalapack) — `scalapack_bench.c`.

Companion to [ELPA](elpa.html): same problem class, different library and parallelism model. ELPA runs one rank with threaded BLAS; ScaLAPACK runs many MPI ranks each with single-threaded BLAS.

## Orange Pi RV2 (SpaceMiT X60, 8 cores)

`na=3000`, `nb=64`, **2×4** grid, 8 ranks × 1 thread. Backends swapped via FlexiBLAS — no rebuild.

| Backend | Time | Result |
| ------- | ---: | ------ |
| Stock CVMFS 0.3.30, default RVV (`ZVL256B`) | — | **HANG** (killed at 200 s) |
| Scalar (`RISCV64_GENERIC`) | 116.87 s | finite=1 |
| Patched RVV (`gemv_n` fix) | **107.23 s** | finite=1 (identical eigenvalues) |

The unpatched vector `gemv_n` NaN does not just corrupt the result — it **hangs** the solver, because `pdsyev`'s serial tridiagonal QR iteration never converges on NaN. The patched build is correct and **1.09×** faster than scalar.

## Why the speedup is modest

Versus ~2.4× for raw `dgemm` and ~1.58× for threaded [ELPA](elpa.html) on the same machine: at 8 ranks the node-local BLAS blocks are small, MPI communication plus the serial tridiagonal solve dominate wall time. The RVV-accelerable BLAS-3 fraction is small — the conservative, communication-bound end of the BLAS-backend spectrum.

## Reproduce

```bash
module load ScaLAPACK/2.2.2-gompi-2025b-fb
make

OMP_NUM_THREADS=1 OPENBLAS_NUM_THREADS=1 OPENBLAS_CORETYPE=RISCV64_GENERIC \
  mpirun --bind-to core -np 8 ./scalapack_bench 3000 64 2

OMP_NUM_THREADS=1 OPENBLAS_NUM_THREADS=1 FLEXIBLAS=/path/to/patched/libopenblas.so \
  mpirun --bind-to core -np 8 ./scalapack_bench 3000 64 2
```

**Toolchain:** ScaLAPACK 2.2.2 / gompi-2025b-fb (EESSI), OpenBLAS 0.3.30 with `gemv_n` fix.
