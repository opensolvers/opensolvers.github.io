# HPL results overview

Cross-board summary of **High Performance Linpack (HPL)** runs on consumer RISC-V hardware through the EESSI stack.

**Toolchain:** GCC 14.3.0, OpenBLAS 0.3.30, HPL 2.3.0. EESSI runs use `2025.06-001` on [`dev.eessi.io/riscv`](https://www.eessi.io/docs/repositories/dev.eessi.io-riscv/).

| Board | Cores | Before | After |
| ----- | ----- | ------ | ----- |
| [StarFive VisionFive 2](../boards/VisionFive2.html) | 4× SiFive U74 | 3.13 GFLOP/s | **5.28 GFLOP/s** |
| [Orange Pi RV2](../boards/RV2.html) | 8× SpacemiT X60 | FAILED (`nan`) | **10.53 GFLOP/s** |
| [Banana Pi F3](../boards/F3.html) | 8× SpacemiT X60 | FAILED (`nan`) | — |

**Before** — stock EESSI OpenBLAS 0.3.30 on each board.

**After** — board-specific fixed OpenBLAS via EasyBuild, swapped in at runtime with FlexiBLAS (no HPL rebuild).

On **VisionFive 2** (scalar U74), stock OpenBLAS uses a generic kernel; the fix adds a tuned U74 micro-kernel ([easyconfigs#26436](https://github.com/easybuilders/easybuild-easyconfigs/pull/26436), [EESSI/docs#818](https://github.com/EESSI/docs/pull/818)).

On **X60 boards** (RVV 1.0, VLEN=256), stock OpenBLAS already dispatches RVV kernels but **0.3.30's `gemv_n` produces NaN**, so HPL fails its residual check despite reporting plausible GFLOP/s. The fix backports the upstream `gemv_n` correction ([easyconfigs#26444](https://github.com/easybuilders/easybuild-easyconfigs/pull/26444), [OpenBLAS#5408](https://github.com/OpenMathLib/OpenBLAS/pull/5408), [EESSI/docs#819](https://github.com/EESSI/docs/pull/819)). Banana Pi F3 uses the same K1/X60 SoC — the same fix applies; peak result not yet recorded.
