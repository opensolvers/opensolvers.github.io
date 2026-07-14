# DGEMM

Micro-benchmarks for **BLAS level-3 performance** and **per-routine correctness** — from [opensolvers/benchmarks/dgemm](https://github.com/opensolvers/benchmarks/tree/main/dgemm). Used to isolate the OpenBLAS 0.3.30 RVV `gemv_n` NaN bug before fixing downstream apps.

See also the [BLAS overview](blas.html) for OpenBLAS packages and board context.

## Tools

| File | Purpose |
| ---- | ------- |
| [`bench_dgemm.c`](https://github.com/opensolvers/benchmarks/blob/main/dgemm/bench_dgemm.c) | Times square `C = A×B` (3 reps), reports GFLOP/s, prints `C[0]` |
| [`difftest.c`](https://github.com/opensolvers/benchmarks/blob/main/dgemm/difftest.c) | `dlopen`s a BLAS `.so`, runs level-1/2/3 routines, reports `sum` / NaN counts |

Both switch backends at runtime via FlexiBLAS or `OPENBLAS_CORETYPE` — no recompile.

```bash
gcc -O2 bench_dgemm.c -o bench_dgemm -lflexiblas
gcc -O2 difftest.c    -o difftest    -ldl -lm

OPENBLAS_NUM_THREADS=8 ./bench_dgemm 4096
./difftest /path/to/libopenblas.so
```

## Correctness — `difftest` (X60, RV2 & F3)

Bit-identical on [Orange Pi RV2](../boards/RV2.html) and [Banana Pi F3](../boards/F3.html):

| Backend | `dgemv` NaN | `dgemm` NaN | `dgemv` sum |
| ------- | ----------- | ----------- | ----------- |
| Stock EESSI, default RVV | **192** | 0 | 198.94 (wrong) |
| Forced scalar | 0 | 0 | 42.06549 (reference) |
| Patched RVV (`gemv_n` fix) | 0 | 0 | 42.06549 (matches) |

Fault is in **`dgemv` only** — plain `dgemm` looks fine on the broken build, which is why [HPL](../apps/hpl.html) can NaN while a GEMM micro-benchmark passes.

## Performance — `bench_dgemm`

### Orange Pi RV2 (1 core, N=2048)

| Backend | GFLOP/s | `C[0]` |
| ------- | ------- | ------ |
| Scalar | 1.16 | 245.24 |
| Patched RVV | 2.62 | 245.24 |

**2.3×** faster, numerically identical.

### Banana Pi F3 (cross-board)

| Backend | GFLOP/s | `C[0]` |
| ------- | ------- | ------ |
| Scalar | 1.26 | 245.24 |
| Patched RVV | 2.96 | 245.24 |

**2.35×** at 1 core. Threaded (8 cores, N=4096): **17.71 GFLOP/s** on patched RVV.
