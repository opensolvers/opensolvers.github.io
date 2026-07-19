# BLAS (OpenBLAS)

Improvements to **OpenBLAS 0.3.30** on RISC-V boards — built via EasyBuild, deployed through [EESSI](https://www.eessi.io/) with **FlexiBLAS** runtime swapping.

Verification microbenchmarks live in [opensolvers/benchmarks/OpenBLAS](https://github.com/opensolvers/benchmarks/tree/main/OpenBLAS) (`bench_dgemm`, `difftest`, `verify_ctrsm`). **Alternative BLAS:** [BLIS](blis.html) RVV vs OpenBLAS DGEMM A/B on X60. Stack probes: [NumPy](numpy.html) (`bench_blas.py`). Full repo: [opensolvers/benchmarks](https://github.com/opensolvers/benchmarks).

**Base stack:** GCC 14.3.0, OpenBLAS 0.3.30, EESSI `2025.06-001` ([`dev.eessi.io/riscv`](https://www.eessi.io/docs/repositories/dev.eessi.io-riscv/)).

## Improvements

| Board / CPU | Problem (stock 0.3.30) | Fix | Result |
| ----------- | ---------------------- | --- | ------ |
| [VisionFive 2](../boards/VisionFive2.html) — SiFive **U74** (scalar) | No U74 kernel; falls back to generic `RISCV64_GENERIC` C `2×2` GEMM | New **4×4 DGEMM micro-kernel** in RV64 assembly, `TARGET=U74` | Single-core DGEMM **~1.4 → 1.77 GFLOP/s**; 4-core DGEMM **6.31 GFLOP/s**; [HPL **1.69×**](../apps/hpl.html) |
| [Orange Pi RV2](../boards/RV2.html) — SpacemiT **X60** (RVV VLEN=256) | RVV `gemv_n` zeroes an **uninitialized** vector register → `dgemv` returns NaN | Backport upstream `gemv_n` fix; `TARGET=RISCV64_ZVL256B` | Verification 2.3×; [NumPy](numpy.html) DGEMM 2.4×; [HPL](../apps/hpl.html) **10.53 GFLOP/s** |
| [Banana Pi F3](../boards/F3.html) — same K1 / X60 SoC | Same RVV `gemv_n` bug as RV2 | Same [easyconfigs#26444](https://github.com/easybuilders/easybuild-easyconfigs/pull/26444) fix | Verification 2.35×; [NumPy](numpy.html) DGEMM **3.6×**; HPL **11.52 GFLOP/s** |

## Packages

| Target | EasyBuild PR | Upstream OpenBLAS | Walkthrough |
| ------ | ------------ | ----------------- | ----------- |
| SiFive U74 | [easyconfigs#26436](https://github.com/easybuilders/easybuild-easyconfigs/pull/26436) | [OpenBLAS#5903](https://github.com/OpenMathLib/OpenBLAS/pull/5903) | [EESSI/docs#818](https://github.com/EESSI/docs/pull/818) |
| SpacemiT X60 | [easyconfigs#26444](https://github.com/easybuilders/easybuild-easyconfigs/pull/26444) | [OpenBLAS#5408](https://github.com/OpenMathLib/OpenBLAS/pull/5408), [#5476](https://github.com/OpenMathLib/OpenBLAS/pull/5476) | [EESSI/docs#819](https://github.com/EESSI/docs/pull/819) · [YouTube](https://www.youtube.com/watch?v=W_-8cKA-CCU) |

Build with `eb --from-pr <num> --robot` into an **EESSI-extend** user install, then `flexiblas add` / `flexiblas default` — no downstream rebuild.

## Verification {#verification}

Small, self-contained programs in [opensolvers/benchmarks/OpenBLAS](https://github.com/opensolvers/benchmarks/tree/main/OpenBLAS) for **BLAS performance and per-routine correctness** — used to isolate broken RVV kernels before trusting downstream apps.

| File | Purpose |
| ---- | ------- |
| [`bench_dgemm.c`](https://github.com/opensolvers/benchmarks/blob/main/OpenBLAS/bench_dgemm.c) | Times square `C = A×B` (3 reps), reports GFLOP/s, prints `C[0]` |
| [`difftest.c`](https://github.com/opensolvers/benchmarks/blob/main/OpenBLAS/difftest.c) | `dlopen`s a BLAS `.so`, runs level-1/2/3 routines, reports `sum` / NaN counts |
| [`verify_ctrsm.c`](https://github.com/opensolvers/benchmarks/blob/main/OpenBLAS/verify_ctrsm.c) | Full-parameter TRSM correctness sweep (localizes [OpenBLAS#5928](https://github.com/OpenMathLib/OpenBLAS/pull/5928) VLEN bug in `_rvv_v1` kernels) |

Both switch backends at runtime via FlexiBLAS or `OPENBLAS_CORETYPE` — no recompile.

```bash
gcc -O2 bench_dgemm.c -o bench_dgemm -lflexiblas
gcc -O2 difftest.c    -o difftest    -ldl -lm

OPENBLAS_NUM_THREADS=8 ./bench_dgemm 4096
./difftest /path/to/libopenblas.so
```

### Correctness — `difftest` (X60, RV2 & F3)

Bit-identical on [Orange Pi RV2](../boards/RV2.html) and [Banana Pi F3](../boards/F3.html):

| Backend | `dgemv` NaN | `dgemm` NaN | `dtrsm` NaN | `dgemv` sum |
| ------- | ----------- | ----------- | ----------- | ----------- |
| Stock EESSI, default RVV | **192** | 0 | 0 | 198.94 (wrong) |
| Forced scalar | 0 | 0 | 0 | 42.06549 (reference) |
| Patched RVV (`gemv_n` fix) | 0 | 0 | 0 | 42.06549 (matches) |

Fault is in **`dgemv` only** — plain `dgemm` and `dtrsm` look fine on the broken `gemv_n` build, which is why [HPL](../apps/hpl.html) and [Quantum ESPRESSO](../apps/qe.html) can fail while a GEMM micro-benchmark passes.

A second bug — RVV `_rvv_v1` TRSM kernels not VLEN-agnostic ([OpenBLAS#5928](https://github.com/OpenMathLib/OpenBLAS/pull/5928)) — is caught by `verify_ctrsm` on `ZVL128B` builds where `GEMM_UNROLL_M ≠ VSETVL_MAX`.

### Performance — `bench_dgemm`

#### Orange Pi RV2 (1 core, N=2048)

| Backend | GFLOP/s | `C[0]` |
| ------- | ------- | ------ |
| Scalar | 1.16 | 245.24 |
| Patched RVV | 2.62 | 245.24 |

**2.3×** faster, numerically identical.

#### Banana Pi F3 (cross-board)

| Backend | GFLOP/s | `C[0]` |
| ------- | ------- | ------ |
| Scalar | 1.26 | 245.24 |
| Patched RVV | 2.96 | 245.24 |

**2.35×** at 1 core. Threaded (8 cores, N=4096): **17.71 GFLOP/s** on patched RVV.

## Notes

- **U74** — performance kernel; stock OpenBLAS works but leaves FP throughput on the table.
- **X60** — correctness fix first. Stock EESSI `DYNAMIC_ARCH` *does* dispatch RVV, but 0.3.30's broken `gemv_n` corrupts BLAS-2 paths. OpenBLAS **≥ 0.3.34** should fix this natively.
- Pin a valid `-march` via `EASYBUILD_OPTARCH` on the experimental `dev.eessi.io/riscv` toolchain (see board walkthroughs).
