---
title: BLIS on RISC-V (X60)
description: BLIS rv64iv vs OpenBLAS RVV on SpaceMiT X60 — 1.29× single-thread DGEMM, weaker HPL; correctness via verify_ctrsm.
---

# BLIS

**Video:** [BLIS vs OpenBLAS on RISC-V](https://www.youtube.com/watch?v=zLMkNrl3NNw) — [all videos](../videos.html)

[BLIS](https://github.com/flame/blis) (FLAME) on SpaceMiT X60 — **vector vs vector** DGEMM against patched RVV [OpenBLAS](blas.html). Both use RVV; BLIS picks hand-written kernels via config **`rv64iv`**.

Harness: [opensolvers/benchmarks/BLIS](https://github.com/opensolvers/benchmarks/tree/main/BLIS). End-to-end: [HPL on BLIS](../apps/hpl.html#hpl-on-blis--end-to-end-validation).

## Headline

| Axis | Result |
| ---- | ------ |
| Square DGEMM, 1 thread, N=4096 (RV2) | BLIS **1.29×** OpenBLAS |
| Square DGEMM, 8 threads | BLIS **0.80–0.89×** (OpenBLAS scales better) |
| HPL (same `libblis.a`) | **PASSED**, but **0.35–0.53×** OpenBLAS-RVV |
| `verify_ctrsm` | **2400/0**; DGEMM `C[0]` identical |

Square 1T wins do **not** predict Linpack — HPL hits skinny rank-k / `dtrsm` / `dgemv` where BLIS trails.

## Why link, not FlexiBLAS

FlexiBLAS is not on the RV2 for this A/B. Same `bench_dgemm.c` is linked once against each library (`-O3 -march=rv64imafdcv_zvl256b`). HPL uses a dedicated `xhpl` + static `libblis.a`.

## Reproduce

```bash
./build-blis.sh                    # rv64iv + OpenMP → $HOME/blis-install
BLIS_PREFIX=$HOME/blis-install \
OPENBLAS_LIB=/path/to/libopenblas.a \
  ./run-ab.sh
```

Use **`rv64iv`** (not `sifive_rvv` / VLEN=128). Enable OpenMP or DGEMM stays ~2.7 GFLOP/s regardless of threads. Prefer EESSI **GCC 14.3** ahead of compat GCC 13.

## DGEMM (Orange Pi RV2)

BLIS `061c2eb` vs patched OpenBLAS, best of 3 reps:

| Threads | N=2048 | N=4096 |
| ------: | -----: | -----: |
| 1 (BLIS / OpenBLAS) | 2.73 / 2.25 (**1.21×**) | 2.95 / 2.28 (**1.29×**) |
| 8 (BLIS / OpenBLAS) | 9.60 / 10.83 (0.89×) | 9.55 / 11.94 (0.80×) |

BPI-F3: same CTRSM pass; 1T N=4096 ~**1.07×** vs stock CVMFS OpenBLAS 0.3.30 — detail in [benchmarks/BLIS](https://github.com/opensolvers/benchmarks/tree/main/BLIS).
