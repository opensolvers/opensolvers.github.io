The Orange Pi RV2 is built on the SpaceMiT **K1** SoC: eight **SpacemiT X60** cores at ~1.6 GHz (`rv64gcv`, **RVV 1.0, VLEN=256**), integrated **IMG BXE-2-32** GPU (OpenCL 3.0 / Vulkan 1.3), 8 GB RAM.

![](rv2.png)

## IME (Integer Matrix Extension)

Besides RVV, each X60 core cluster exposes SpaceMiT's **IME** — a dedicated **int8 matrix unit** via the custom instruction `smt.vmadot`. One `vmadot` fuses a **4×4 int32** tile update from two **4×8 int8** operand tiles; this is the hardware behind the part's quoted AI TOPS rating.

On the K1 the IME sits in **cluster 0 only** (cores **0–3**), with those four cores sharing a **512 KB L2**. Pin IME workloads to a cluster-0 core (`taskset -c 0`).

Microbenchmarks in [opensolvers/benchmarks/ime](https://github.com/opensolvers/benchmarks/tree/main/ime) (`ime-bench`): pure `s8s8s32` GEMM, bit-exact vs a scalar reference, timed against a plain RVV int8 baseline on this board (core 0, 1.6 GHz):

| M×N×K | RVV int8 | IME (`smt.vmadot`) | IME / RVV |
| ----- | -------- | ------------------ | --------- |
| 512×512×512 | 5.2 GOP/s | **39 GOP/s** | **7.5×** |
| 768×768×512 | ~5.2 GOP/s | **42 GOP/s** (peak) | **8.1×** |
| 1024×1024×512 | 5.2 GOP/s | **32 GOP/s** | 6.2× |

Peak **~42 GOP/s** single-core — vs ~5 GOP/s for a straightforward RVV int8 path. End-to-end int4 LLM decode through [ONNX Runtime](../apps/onnx.html) and isolated [MLAS](../scientific-libs/mlas.html) kernel rates use the same IME hardware; see also [papers/x60-ime-block-scale-optimization](https://github.com/opensolvers/benchmarks/blob/main/papers/x60-ime-block-scale-optimization.md) in the benchmarks repo.

## HPL via EESSI

See also the [HPL app overview](../apps/hpl.html).

Unlike the scalar VisionFive 2, the X60 already dispatches OpenBLAS's upstream **RVV** `RISCV64_ZVL256B` kernels from the stock EESSI stack — but OpenBLAS **0.3.30** has a bug in `gemv_n` that zeroes an uninitialized vector register, so stock EESSI **HPL fails with residual `nan`** (it can still report a plausible ~8.5 GFLOP/s; only the residual check reveals the answer is wrong).

End-to-end on real Orange Pi RV2 hardware using [EESSI](https://www.eessi.io/) `2025.06-001` on [`dev.eessi.io/riscv`](https://www.eessi.io/docs/repositories/dev.eessi.io-riscv/). Peak run: **N=20000**, **NB=256**, **2×4** grid (8 MPI ranks).

| | Before | After |
| --- | ------ | ----- |
| HPL (8 cores, N=20000, 2×4) | ~8.5 GFLOP/s, **FAILED** (`nan`) | **10.53 GFLOP/s**, **PASSED** |
| Residual (N=8000, 1×8) | `nan` | 4.04e-03 |

With the fixed backend, scalar-vs-RVV A/B ([benchmarks/hpl](https://github.com/opensolvers/benchmarks/tree/main/hpl)): **6.41 → 11.55 GFLOP/s** (N=8000, 1×8) and **7.38 → 13.41 GFLOP/s** (N=28672, 1×8).

**Before** — stock EESSI OpenBLAS 0.3.30 (RVV `gemv_n` bug). **After** — fixed OpenBLAS built with `TARGET=RISCV64_ZVL256B` and a backported `gemv_n` patch ([easyconfigs#26444](https://github.com/easybuilders/easybuild-easyconfigs/pull/26444)), swapped in via FlexiBLAS — no HPL rebuild.

The fix backports the upstream `gemv_n` correction from OpenBLAS ≥ 0.3.31 ([OpenBLAS#5408](https://github.com/OpenMathLib/OpenBLAS/pull/5408)). A future EESSI bump to OpenBLAS ≥ 0.3.34 should make the patch unnecessary.

### Reproducing the fixed run

1. Set up CVMFS + EESSI on `riscv64` (`EESSI_VERSION_OVERRIDE=2025.06-001`).
2. Baseline: `module load HPL/2.3-foss-2025b` → stock HPL fails residual check (`nan`).
3. Build fixed OpenBLAS: `eb --from-pr 26444 --robot` (via `EESSI-extend` user install, `EASYBUILD_OPTARCH='-march=rv64imafdcv_zvl256b'`).
4. Register the new backend with FlexiBLAS and re-run the same `xhpl`.

Full walkthrough: [EESSI/docs#819](https://github.com/EESSI/docs/pull/819) — *Chasing a NaN: correct RVV HPL on a RISC-V SpaceMiT X60 via EESSI*.

## FFTW RVV

See [FFTW](../scientific-libs/fftw.html) — r5v wins **1.06–1.60×** in `tests/bench`, but **~0%** end-to-end in [Quantum ESPRESSO](../apps/qe.html) (`FFTW_ESTIMATE`). [GROMACS](../apps/gromacs.html) FFT swap: **1.23×** on isolated `PME 3D-FFT`.

## ScaLAPACK

See [ScaLAPACK](../scientific-libs/scalapack.html) — `PDSYEV` on 2×4 grid: stock RVV **hangs**; patched RVV **107.23 s** vs scalar **116.87 s** (**1.09×**).
