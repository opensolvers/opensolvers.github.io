# NumPy

A **SciPy-stack probe** for BLAS and LAPACK through NumPy — from [opensolvers/benchmarks/numpy](https://github.com/opensolvers/benchmarks/tree/main/numpy) (`bench_blas.py`).

| Kernel | NumPy call | Backend | Metric |
| ------ | ---------- | ------- | ------ |
| DGEMM | `A @ B` | `dgemm` | GFLOP/s |
| EIGH | `np.linalg.eigvalsh(S)` | `dsyevd` | seconds |

Unlike raw [OpenBLAS verification](blas.html#verification), `eigvalsh` exercises BLAS-2 tridiagonalization — a correctness gate for the same `gemv_n` bug that breaks [HPL](../apps/hpl.html) and [ELPA](elpa.html).

```bash
module load SciPy-bundle/2025.07-gfbf-2025b
OMP_NUM_THREADS=8 OPENBLAS_NUM_THREADS=8 python3 bench_blas.py [dgemm_N] [eigh_N]
# defaults: dgemm_N=4096  eigh_N=2048
```

Swap backends per run: `OPENBLAS_CORETYPE=RISCV64_GENERIC` or `FLEXIBLAS=/path/to/libopenblas.so`.

## Orange Pi RV2 (8 threads, patched RVV)

| Kernel | Scalar | Patched RVV | Speedup |
| ------ | ------ | ----------- | ------- |
| DGEMM N=4096 | 4.77 GFLOP/s | 11.52 GFLOP/s | **2.4×** |
| EIGH N=2048 | 10.54 s | 6.72 s | **1.6×** |

Both patched results finite. See [LAPACK](lapack.html) for the LAPACK angle.

## Banana Pi F3 (cross-board, 8 threads)

| Kernel | Scalar | Patched RVV | Speedup |
| ------ | ------ | ----------- | ------- |
| DGEMM N=4096 | 4.91 GFLOP/s | **17.51 GFLOP/s** | **3.6×** |
| EIGH N=2048 | 9.59 s | 5.94 s | **1.6×** |

**Stock unpatched RVV:** `eigvalsh` raises `LinAlgError: Eigenvalues did not converge`; DGEMM alone stays finite at 10.94 GFLOP/s (only `gemv` is broken — see [`difftest`](blas.html#verification)).

Eigensolver speedup (~1.6×) is lower than DGEMM (~2.4–3.6×) because `eigvalsh` mixes BLAS-3 with latency-bound BLAS-2 — same pattern as [ELPA](elpa.html).
