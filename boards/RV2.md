The Orange Pi RV2 is built on the SpaceMiT **K1** SoC: eight **SpacemiT X60** cores at ~1.6 GHz (`rv64gcv`, **RVV 1.0, VLEN=256**), 8 GB RAM.

![](rv2.png)

## HPL via EESSI

See also the [HPL app overview](../apps/hpl.html).

Unlike the scalar VisionFive 2, the X60 already dispatches OpenBLAS's upstream **RVV** `RISCV64_ZVL256B` kernels from the stock EESSI stack — but OpenBLAS **0.3.30** has a bug in `gemv_n` that zeroes an uninitialized vector register, so stock EESSI **HPL fails with residual `nan`** (it can still report a plausible ~8.5 GFLOP/s; only the residual check reveals the answer is wrong).

End-to-end on real Orange Pi RV2 hardware using [EESSI](https://www.eessi.io/) `2025.06-001` on [`dev.eessi.io/riscv`](https://www.eessi.io/docs/repositories/dev.eessi.io-riscv/). Peak run: **N=20000**, **NB=256**, **2×4** grid (8 MPI ranks).

| | Before | After |
| --- | ------ | ----- |
| HPL (8 cores, N=20000, 2×4) | ~8.5 GFLOP/s, **FAILED** (`nan`) | **10.53 GFLOP/s**, **PASSED** |
| Residual (N=8000, 1×8) | `nan` | 4.04e-03 |

**Before** — stock EESSI OpenBLAS 0.3.30 (RVV `gemv_n` bug). **After** — fixed OpenBLAS built with `TARGET=RISCV64_ZVL256B` and a backported `gemv_n` patch ([easyconfigs#26444](https://github.com/easybuilders/easybuild-easyconfigs/pull/26444)), swapped in via FlexiBLAS — no HPL rebuild.

The fix backports the upstream `gemv_n` correction from OpenBLAS ≥ 0.3.31 ([OpenBLAS#5408](https://github.com/OpenMathLib/OpenBLAS/pull/5408)). A future EESSI bump to OpenBLAS ≥ 0.3.34 should make the patch unnecessary.

### Reproducing the fixed run

1. Set up CVMFS + EESSI on `riscv64` (`EESSI_VERSION_OVERRIDE=2025.06-001`).
2. Baseline: `module load HPL/2.3-foss-2025b` → stock HPL fails residual check (`nan`).
3. Build fixed OpenBLAS: `eb --from-pr 26444 --robot` (via `EESSI-extend` user install, `EASYBUILD_OPTARCH='-march=rv64imafdcv_zvl256b'`).
4. Register the new backend with FlexiBLAS and re-run the same `xhpl`.

Full walkthrough: [EESSI/docs#819](https://github.com/EESSI/docs/pull/819) — *Chasing a NaN: correct RVV HPL on a RISC-V SpaceMiT X60 via EESSI*.
