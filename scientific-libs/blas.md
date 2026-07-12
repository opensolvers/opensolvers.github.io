# BLAS (OpenBLAS)

Improvements to **OpenBLAS 0.3.30** on RISC-V boards — built via EasyBuild, deployed through [EESSI](https://www.eessi.io/) with **FlexiBLAS** runtime swapping. Verification tools and A/B scripts live in the [opensolvers/benchmarks](https://github.com/opensolvers/benchmarks) repo (`dgemm/`, `numpy/`).

**Base stack:** GCC 14.3.0, OpenBLAS 0.3.30, EESSI `2025.06-001` ([`dev.eessi.io/riscv`](https://www.eessi.io/docs/repositories/dev.eessi.io-riscv/)).

## Improvements

| Board / CPU | Problem (stock 0.3.30) | Fix | Result |
| ----------- | ---------------------- | --- | ------ |
| [VisionFive 2](../boards/VisionFive2.html) — SiFive **U74** (scalar) | No U74 kernel; falls back to generic `RISCV64_GENERIC` C `2×2` GEMM | New **4×4 DGEMM micro-kernel** in RV64 assembly, `TARGET=U74` | Single-core DGEMM **~1.4 → 1.77 GFLOP/s**; 4-core DGEMM **6.31 GFLOP/s**; [HPL **1.69×**](../apps/hpl.html) |
| [Orange Pi RV2](../boards/RV2.html) — SpacemiT **X60** (RVV VLEN=256) | RVV `gemv_n` zeroes an **uninitialized** vector register → `dgemv` returns NaN | Backport upstream `gemv_n` fix; `TARGET=RISCV64_ZVL256B` | Correct `dgemv`; see [HPL](../apps/hpl.html) and A/B tables below |
| [Banana Pi F3](../boards/F3.html) — same K1 / X60 SoC | Same RVV `gemv_n` bug as RV2 | Same [easyconfigs#26444](https://github.com/easybuilders/easybuild-easyconfigs/pull/26444) fix | Fix applies; peak results not yet recorded |

## X60 A/B — patched RVV vs scalar (Orange Pi RV2)

All runs use a **fixed** RVV OpenBLAS (`gemv_n` patch) swapped in via FlexiBLAS. Source: [benchmarks/dgemm](https://github.com/opensolvers/benchmarks/tree/main/dgemm), [benchmarks/numpy](https://github.com/opensolvers/benchmarks/tree/main/numpy).

| Probe | Scalar (`RISCV64_GENERIC`) | Patched RVV (`ZVL256B`) | Speedup |
| ----- | -------------------------- | ----------------------- | ------- |
| `bench_dgemm` N=2048, 1 core | 1.16 GFLOP/s | 2.62 GFLOP/s | **2.3×** |
| NumPy `A @ B` N=4096, 8 threads | 4.77 GFLOP/s | 11.52 GFLOP/s | **2.4×** |
| NumPy `eigvalsh` N=2048, 8 threads | 10.54 s | 6.72 s | **1.6×** |

## X60 — isolating the `gemv_n` bug (`difftest`)

The [benchmarks `difftest`](https://github.com/opensolvers/benchmarks/blob/main/dgemm/difftest.c) tool `dlopen`s a BLAS `.so` and reports NaN counts per routine:

| Backend | `dgemv` NaN | `dgemm` NaN | `dgemv` sum |
| ------- | ----------- | ----------- | ----------- |
| Stock EESSI, default RVV | **192** | 0 | 198.94 (wrong) |
| Stock EESSI, forced scalar | 0 | 0 | 42.06549 (reference) |
| Patched RVV (`gemv_n` fix) | 0 | 0 | 42.06549 (matches) |

`dgemm` looks fine on the broken build — which is why a raw GEMM benchmark can pass while [HPL](../apps/hpl.html) and [ELPA](elpa.html) fail.

## Packages

| Target | EasyBuild PR | Upstream OpenBLAS | Walkthrough |
| ------ | ------------ | ----------------- | ----------- |
| SiFive U74 | [easyconfigs#26436](https://github.com/easybuilders/easybuild-easyconfigs/pull/26436) | [OpenBLAS#5903](https://github.com/OpenMathLib/OpenBLAS/pull/5903) | [EESSI/docs#818](https://github.com/EESSI/docs/pull/818) |
| SpacemiT X60 | [easyconfigs#26444](https://github.com/easybuilders/easybuild-easyconfigs/pull/26444) | [OpenBLAS#5408](https://github.com/OpenMathLib/OpenBLAS/pull/5408), [#5476](https://github.com/OpenMathLib/OpenBLAS/pull/5476) | [EESSI/docs#819](https://github.com/EESSI/docs/pull/819) |

Build with `eb --from-pr <num> --robot` into an **EESSI-extend** user install, then `flexiblas add` / `flexiblas default` — no downstream rebuild.

## Notes

- **U74** — performance kernel; stock OpenBLAS works but leaves FP throughput on the table.
- **X60** — correctness fix first. Stock EESSI `DYNAMIC_ARCH` *does* dispatch RVV, but 0.3.30's broken `gemv_n` corrupts BLAS-2 paths. OpenBLAS **≥ 0.3.34** should fix this natively.
- Pin a valid `-march` via `EASYBUILD_OPTARCH` on the experimental `dev.eessi.io/riscv` toolchain (see board walkthroughs).
