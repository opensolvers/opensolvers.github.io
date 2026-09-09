---
title: BLAS (OpenBLAS) on RISC-V
description: OpenBLAS on RISC-V — U74 4×4 DGEMM, X60 gemv_n NaN fix, 0.3.34 verify, FlexiBLAS swaps via EESSI.
---

# BLAS (OpenBLAS)

OpenBLAS on consumer RISC-V boards via [EESSI](https://www.eessi.io/) and **FlexiBLAS** (swap backends without rebuilding apps).

Harness: [opensolvers/benchmarks/OpenBLAS](https://github.com/opensolvers/benchmarks/tree/main/OpenBLAS). Related: [BLIS](blis.html), [NumPy](numpy.html), [HPL](../apps/hpl.html). **Video:** [NaN Linpack on RISC-V](https://www.youtube.com/watch?v=W_-8cKA-CCU) · [EESSI blog](https://www.eessi.io/docs/blog/2026/07/12/risc-v-x60-openblas-hpl/).

## What broke / what we fixed

| Board | Stock problem | Fix | Headline |
| ----- | ------------- | --- | -------- |
| [VisionFive 2](../boards/VisionFive2.html) (U74) | Generic `2×2` GEMM only | **4×4 DGEMM** asm (`TARGET=U74`) | HPL **3.13 → 5.28 GFLOP/s** (**1.69×**) |
| [Orange Pi RV2](../boards/RV2.html) (X60 RVV) | RVV `gemv_n` → **NaN** | Backport `gemv_n` fix | HPL **FAILED → 10.53 GFLOP/s**; DGEMM **2.3×** vs scalar |
| [Banana Pi F3](../boards/F3.html) (same K1) | Same `gemv_n` bug | Same fix | HPL **11.52 GFLOP/s**; DGEMM **2.35×** |

Fault is in **`dgemv` only** — plain `dgemm` can look fine on a broken build, which is why HPL / QE / PETSc fail while a GEMM microbench passes.

## OpenBLAS 0.3.34 (RV2)

Upstream `v0.3.34` (`TARGET=RISCV64_ZVL256B`) has a working RVV path on X60 — verified with [`run-034-tests.sh`](https://github.com/opensolvers/benchmarks/blob/main/OpenBLAS/run-034-tests.sh):

| Check | Result |
| ----- | ------ |
| `dgemv` NaN | **0** (stock 0.3.30: **768**) |
| SYRK PSD / CTRSM | **PASS** / **2400/0** |
| DGEMM N=2048×8 thr | **15.54** GFLOP/s (~1.6× stock 0.3.30) |
| HPL N=8000 / N=20000 | **11.04** / **10.97** GFLOP/s PASSED |

EESSI still ships 0.3.29/0.3.30; use a local 0.3.34 + FlexiBLAS until the stack bumps.

## Packages

| Target | EasyBuild | Upstream |
| ------ | --------- | -------- |
| U74 | [easyconfigs#26436](https://github.com/easybuilders/easybuild-easyconfigs/pull/26436) | [OpenBLAS#5903](https://github.com/OpenMathLib/OpenBLAS/pull/5903) |
| X60 | [easyconfigs#26444](https://github.com/easybuilders/easybuild-easyconfigs/pull/26444) | [OpenBLAS#5408](https://github.com/OpenMathLib/OpenBLAS/pull/5408), [#5476](https://github.com/OpenMathLib/OpenBLAS/pull/5476) |

`eb --from-pr <num> --robot` into EESSI-extend, then `flexiblas add` / `flexiblas default`.

## Reproduce

```bash
gcc -O2 bench_dgemm.c -o bench_dgemm -lflexiblas
gcc -O2 difftest.c -o difftest -ldl -lm
OPENBLAS_NUM_THREADS=8 ./bench_dgemm 4096
./difftest /path/to/libopenblas.so
```

| Tool | Role |
| ---- | ---- |
| `bench_dgemm` | Square GEMM GFLOP/s + `C[0]` |
| `difftest` | Level-1/2/3 sums + NaN counts |
| `verify_ctrsm` | TRSM parameter sweep (VLEN bugs) |
