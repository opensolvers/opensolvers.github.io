---
title: MetalWalls on SpaceMiT X60
description: MetalWalls tip4p-water on Orange Pi RV2 — FFT r5v and FlexiBLAS A/Bs (~1.00–1.02×; long-range only ~11% of wall).
---

# MetalWalls

[MetalWalls](https://gitlab.com/ampere2/metalwalls) is a constant-potential electrochemical molecular-dynamics code (TIP4P / electrode capacitors). On the Orange Pi RV2 we A/B **FFTW** (`LD_PRELOAD`) and **FlexiBLAS** backends under one unchanged `mw` binary.

Benchmark source: [opensolvers/benchmarks/metalwalls](https://github.com/opensolvers/benchmarks/tree/main/metalwalls). Module: `MetalWalls/21.06.1-foss-2023b` on EESSI `riscv.eessi.io` **20240402**.

> **Bottom line:** on upstream **`example/tip4p-water`** (215 molecules, no electrodes), FFT r5v ≈ **1.02×** and BLAS backends ≈ **1.00×**. Long-range Ewald/FFT is only ~**11%** of wall; Coulomb short-range (~50%) and vdW (~29%) dominate. Electrode capacitor examples deferred (too slow for a short A/B).

---

## Profile (tip4p-water)

| Bucket | ≈ % wall |
| ------ | -------: |
| Coulomb short-range | ~50% |
| van der Waals | ~29% |
| Coulomb long-range (Ewald/FFT) | ~11% |
| RATTLE | ~7% |

---

## Results (2026-08-21)

200 MD steps, `mpirun -np 1`. Temperatures match bit-for-bit across runs.

| Tag | Axis | Wall s |
| --- | ---- | -----: |
| fft_scalar | LD_PRELOAD scalar `libfftw3` | 41.27 |
| fft_r5v | LD_PRELOAD r5v `libfftw3` | 40.59 (**~1.02×**) |
| blas_scalar | `OPENBLAS_CORETYPE=RISCV64_GENERIC` | 40.83 |
| blas_stock | default FlexiBLAS OpenBLAS | 40.53 |
| blas_patched | patched RVV OpenBLAS | 40.99 |

**Measured:** 2026-08-21 on Orange Pi RV2.
