# HPL results overview

Cross-board summary of **High Performance Linpack (HPL)** runs on consumer RISC-V hardware, comparing `-march` targets and EESSI stack builds.

**Toolchain:** GCC 14.3.0, OpenBLAS 0.3.30, HPL 2.3.0. EESSI runs use `2025.06-001` on [`dev.eessi.io/riscv`](https://www.eessi.io/docs/repositories/dev.eessi.io-riscv/).

| Board | Cores | Before | After |
| ----- | ----- | ------ | ----- |
| [StarFive VisionFive 2](VisionFive2.html) | 4× SiFive U74 | 3.13 GFLOP/s | **5.28 GFLOP/s** |
| [OrangePi RV2](RV2.html) | 8× SpacemiT X60 | DNF | 7.38 GFLOP/s |
| [BananaPi F3](F3.html) | 8× SpacemiT X60 | DNF | — |

**DNF** — did not finish on the EESSI RISC-V stack, due to an RVV bug in OpenBLAS 0.3.30.

The VisionFive 2 **After** result uses a SiFive U74 OpenBLAS micro-kernel built via EasyBuild, swapped in at runtime with FlexiBLAS — **1.69×** faster than the stock EESSI **Before** build on the same hardware. See the [VisionFive 2 page](VisionFive2.html) and [EESSI/docs#818](https://github.com/EESSI/docs/pull/818) for the full walkthrough.
