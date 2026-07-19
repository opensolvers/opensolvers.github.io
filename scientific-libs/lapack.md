# LAPACK

LAPACK performance and correctness probed via NumPy **`eigvalsh`** (`dsyevd`) — the LAPACK-heavy half of the [NumPy benchmark](numpy.html).

Benchmark source: [opensolvers/benchmarks/numpy](https://github.com/opensolvers/benchmarks/tree/main/numpy) (`bench_blas.py`). For DGEMM (`A @ B`) see the [NumPy](numpy.html) or [BLAS verification](blas.html#verification) sections.

## Orange Pi RV2 (8 threads)

Patched RVV OpenBLAS via FlexiBLAS (`SciPy-bundle` module, [easyconfigs#26444](https://github.com/easybuilders/easybuild-easyconfigs/pull/26444)).

| Kernel | Scalar | Patched RVV | Speedup |
| ------ | ------ | ----------- | ------- |
| EIGH (`eigvalsh`, N=2048) | 10.54 s | 6.72 s | **1.6×** |

Stock EESSI default RVV: `eigvalsh` returns NaN / fails to converge. See [`difftest`](blas.html#verification).

## Banana Pi F3 (cross-board)

| Kernel | Scalar | Patched RVV | Speedup |
| ------ | ------ | ----------- | ------- |
| EIGH N=2048 | 9.59 s | 5.94 s | **1.6×** |

Stock unpatched RVV: `LinAlgError: Eigenvalues did not converge`.

For the full NumPy benchmark (DGEMM + EIGH), see [NumPy](numpy.html). For a heavier eigensolver probe, see [ELPA](elpa.html).
