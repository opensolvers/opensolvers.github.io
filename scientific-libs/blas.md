# BLAS (OpenBLAS)

Improvements to **OpenBLAS 0.3.30** on RISC-V boards — built via EasyBuild, deployed through [EESSI](https://www.eessi.io/) with **FlexiBLAS** runtime swapping.

Verification and A/B probes: [DGEMM](dgemm.html) (`bench_dgemm`, `difftest`), [NumPy](numpy.html) (`bench_blas.py`). Source repo: [opensolvers/benchmarks](https://github.com/opensolvers/benchmarks).

**Base stack:** GCC 14.3.0, OpenBLAS 0.3.30, EESSI `2025.06-001` ([`dev.eessi.io/riscv`](https://www.eessi.io/docs/repositories/dev.eessi.io-riscv/)).

## Improvements

| Board / CPU | Problem (stock 0.3.30) | Fix | Result |
| ----------- | ---------------------- | --- | ------ |
| [VisionFive 2](../boards/VisionFive2.html) — SiFive **U74** (scalar) | No U74 kernel; falls back to generic `RISCV64_GENERIC` C `2×2` GEMM | New **4×4 DGEMM micro-kernel** in RV64 assembly, `TARGET=U74` | Single-core DGEMM **~1.4 → 1.77 GFLOP/s**; 4-core DGEMM **6.31 GFLOP/s**; [HPL **1.69×**](../apps/hpl.html) |
| [Orange Pi RV2](../boards/RV2.html) — SpacemiT **X60** (RVV VLEN=256) | RVV `gemv_n` zeroes an **uninitialized** vector register → `dgemv` returns NaN | Backport upstream `gemv_n` fix; `TARGET=RISCV64_ZVL256B` | [DGEMM](dgemm.html) 2.3×; [NumPy](numpy.html) DGEMM 2.4×; [HPL](../apps/hpl.html) **10.53 GFLOP/s** |
| [Banana Pi F3](../boards/F3.html) — same K1 / X60 SoC | Same RVV `gemv_n` bug as RV2 | Same [easyconfigs#26444](https://github.com/easybuilders/easybuild-easyconfigs/pull/26444) fix | [DGEMM](dgemm.html) 2.35×; [NumPy](numpy.html) DGEMM **3.6×**; HPL **11.52 GFLOP/s** |

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
