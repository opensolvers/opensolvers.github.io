# LAPACK

LAPACK correctness and performance here are probed through **NumPy** (`eigvalsh` → `dsyevd`) and pure BLAS (`A @ B` → `dgemm`) — a real-application path for SciPy/NumPy stacks on EESSI.

Benchmark source: [opensolvers/benchmarks/numpy](https://github.com/opensolvers/benchmarks/tree/main/numpy).

## Orange Pi RV2 (SpaceMiT X60, 8 threads)

Backends swapped via FlexiBLAS on a NumPy linked against the FlexiBLAS hub (`SciPy-bundle` module). Patched RVV OpenBLAS uses the `gemv_n` fix from [easyconfigs#26444](https://github.com/easybuilders/easybuild-easyconfigs/pull/26444).

| Kernel | Scalar (`RISCV64_GENERIC`) | Patched RVV (`ZVL256B`) | Speedup |
| ------ | -------------------------- | ----------------------- | ------- |
| DGEMM (`A @ B`, N=4096) | 4.77 GFLOP/s | 11.52 GFLOP/s | **2.4×** |
| EIGH (`eigvalsh`, N=2048) | 10.54 s | 6.72 s | **1.6×** |

Both patched runs are **finite** (correct through real LAPACK). The eigensolver speedup is lower than DGEMM because `eigvalsh` mixes BLAS-3 with BLAS-2 tridiagonalization — the same pattern seen in [ELPA](elpa.html).

Stock EESSI default RVV would return NaN on the eigensolver path; see the [BLAS `difftest` table](blas.html).

### Banana Pi BPI-F3 (cross-board confirmation)

| Kernel | Scalar | Patched RVV | Speedup |
| ------ | ------ | ----------- | ------- |
| DGEMM N=4096 | 4.91 GFLOP/s | **17.51 GFLOP/s** | **3.6×** |
| EIGH N=2048 | 9.59 s | 5.94 s | **1.6×** |

Stock unpatched RVV: `eigvalsh` aborts with `LinAlgError: Eigenvalues did not converge` (DGEMM alone stays finite at 10.94 GFLOP/s).
