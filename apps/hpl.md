---
title: HPL results on RISC-V boards
description: High Performance Linpack cross-board summary — VisionFive 2 U74 tuning, Orange Pi RV2 and Banana Pi F3 X60 fixes, EESSI stack, and FlexiBLAS backend swaps.
---

# HPL results overview

Cross-board summary of **High Performance Linpack (HPL)** on consumer RISC-V hardware through the EESSI stack. Configs and an A/B runner (`run-hpl-ab.sh`) are in [opensolvers/benchmarks/hpl](https://github.com/opensolvers/benchmarks/tree/main/hpl).

**Toolchain:** GCC 14.3.0, OpenBLAS 0.3.30, HPL 2.3.0. EESSI `2025.06-001` on [`dev.eessi.io/riscv`](https://www.eessi.io/docs/repositories/dev.eessi.io-riscv/).

## Cross-board summary

| Board | Cores | Before | After |
| ----- | ----- | ------ | ----- |
| [VisionFive 2](../boards/VisionFive2.html) | 4× SiFive U74 | 3.13 GFLOP/s | **5.28 GFLOP/s** |
| [Orange Pi RV2](../boards/RV2.html) | 8× SpacemiT X60 | FAILED (`nan`) | **10.53 GFLOP/s** |
| [Banana Pi F3](../boards/F3.html) | 8× SpacemiT X60 | FAILED (`nan`) | **11.52 GFLOP/s** |

**Before** — stock EESSI OpenBLAS 0.3.30 (same `xhpl` binary throughout).

**After** — fixed OpenBLAS via EasyBuild + FlexiBLAS backend swap (no HPL rebuild). See [BLAS overview](../scientific-libs/blas.html).

## Orange Pi RV2 — detailed results

**Video:** [NaN Linpack on RISC-V: Fixing OpenBLAS gemv_n on Orange Pi RV2 (EESSI)](https://www.youtube.com/watch?v=W_-8cKA-CCU) — [all videos](../videos.html)

Stock EESSI dispatches RVV `ZVL256B` on the X60, but the unpatched `gemv_n` kernel makes HPL report ~8.5 GFLOP/s while **failing** the residual check (`nan`). With the [easyconfigs#26444](https://github.com/easybuilders/easybuild-easyconfigs/pull/26444) fix, all runs below **PASSED**.

### EESSI walkthrough (fixed backend)

| Config | Grid | N | Result |
| ------ | ---- | - | ------ |
| Stock EESSI, default RVV | 1×8 | 8000 | ~8.5 GFLOP/s, **FAILED** (`nan`) |
| Fixed RVV, peak | 2×4 | 20000 | **10.53 GFLOP/s**, PASSED |

Walkthrough: [EESSI/docs#819](https://github.com/EESSI/docs/pull/819).

### A/B — scalar vs patched RVV (`run-hpl-ab.sh`)

Same `xhpl`, backend swapped via FlexiBLAS. All **PASSED** with the fixed vector library:

| Config | Scalar (`RISCV64_GENERIC`) | Patched RVV (`ZVL256B`) | Speedup |
| ------ | -------------------------- | ----------------------- | ------- |
| `HPL.dat` (N=8000, 1×8) | 6.41 GFLOP/s | 11.55 GFLOP/s | **1.80×** |
| `HPL_big.dat` (N=28672, 1×8) | 7.38 GFLOP/s | 13.41 GFLOP/s | **1.82×** |
| `HPL-sweep.dat` (N=20000, 2×4) | — | ~10.5 GFLOP/s | — |

A squarer **2×4** grid beats 1×8 for peak throughput. `HPL_big.dat` needs ~6.6 GB RAM — tight on 8 GB boards.

### HPL on BLIS — end-to-end validation

Unlike the FlexiBLAS OpenBLAS A/B above, BLIS cannot be swapped at runtime on the RV2 (no FlexiBLAS). A dedicated `xhpl` is linked statically against RVV `libblis.a` (`Make.rv64_blis`, `build-hpl-blis.sh`). Source: [benchmarks/hpl](https://github.com/opensolvers/benchmarks/tree/main/hpl).

This validates that RVV BLIS drives a real Linpack solve — not just square DGEMM — because HPL's panel path leans on `dgemv`/`dtrsv` (the same reason stock OpenBLAS NaN'd).

| Config | BLIS GFLOP/s | Residual | vs patched OpenBLAS-RVV |
| ------ | -----------: | -------- | ----------------------: |
| `HPL.dat` (N=8000, 1×8) | 4.02 | 4.12e-03 PASSED | **0.35×** |
| `HPL-sweep.dat` (N=20000, 2×4) | 5.57 | 3.39e-03 PASSED | **0.53×** |

#### Full-memory process-grid sweep (N=25600)

Largest square problem that fits (~5.2 GB). N=28672 OOMs on this 8 GB no-swap board (rank 7 signal 9).

| Grid | GFLOP/s | Time | Residual |
| ---- | ------: | ---: | -------- |
| **2×4** | **5.87** | 1904.5 s | 3.37e-03 PASSED |
| 1×8 | 5.84 | 1916.0 s | 2.84e-03 PASSED |
| 4×2 | 5.23 | 2140.5 s | 4.07e-03 PASSED |
| 8×1 | 4.31 | 2592.9 s | 5.75e-03 PASSED |

**Takeaway:** correctness holds (no NaN, residual on par with OpenBLAS). Throughput trails patched-RVV OpenBLAS — HPL is panel-heavy (thin k=256, `dtrsm`, level-2), not the large square DGEMM where BLIS wins ~1.2–1.3× single-thread. Wide grids beat tall (8×1 −26% vs 2×4). See [BLIS](../scientific-libs/blis.html).

## VisionFive 2

Scalar U74 — stock OpenBLAS uses a generic kernel; the U74-tuned build lifts HPL **3.13 → 5.28 GFLOP/s** (**1.69×**). Walkthrough: [EESSI/docs#818](https://github.com/EESSI/docs/pull/818).

## Banana Pi F3

Same K1 / X60 SoC as the Orange Pi RV2 — cross-board confirmation on [opensolvers/benchmarks](https://github.com/opensolvers/benchmarks). **3.7 GB RAM** limits problem size: only `HPL.dat` (N=8000) was run; larger configs need more memory than this board has.

| Backend | GFLOP/s | Residual | Result |
| ------- | ------- | -------- | ------ |
| Stock EESSI RVV | 11.64 | `nan` | FAILED |
| Scalar | 6.52 | 4.63e-03 | PASSED |
| Patched RVV | **11.52** | 4.04e-03 | PASSED |

**1.77×** scalar → patched vector; residual bit-identical to RV2.
