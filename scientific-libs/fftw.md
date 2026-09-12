---
title: FFTW r5v on RISC-V (X60)
description: FFTW 3.3.10 RVV r5v vs scalar — up to 1.60× micro under MEASURE; ~0% in QE ESTIMATE; wisdom ~6%; planner beats codelets.
---

# FFTW

**Video:** [When RVV FFT Wins 1.6× — and Gives 0% in Quantum ESPRESSO](https://www.youtube.com/watch?v=FlumrCEUIBE) — [all videos](../videos.html)

[FFTW](https://www.fftw.org/) 3.3.10 with the **RISC-V Vector (`r5v`) SIMD backend** vs a scalar build of the same source. Only flag: `--enable-r5v` ([rdolbeau `r5v-test-release-005`](https://github.com/rdolbeau)).

Harness: [opensolvers/benchmarks/fftw](https://github.com/opensolvers/benchmarks/tree/main/fftw). Swap via `LD_PRELOAD` (not FlexiBLAS). Apps: [QE](../apps/qe.html), [GROMACS](../apps/gromacs.html).

## Headline

| Axis | Result |
| ---- | ------ |
| Microbench (`FFTW_MEASURE`, N=256) | r5v **1.60×** scalar |
| Same, large N (≥64K) | **~1.06×** (bandwidth-bound) |
| Planner ESTIMATE → MEASURE | **3–5×** (often bigger than RVV) |
| QE 64-atom SCF (stock ESTIMATE) | **~0%** wall / **~1.9%** in `fftw` |
| QE + MEASURE wisdom | **~6%** wall (serial); MPI ~2–6% |
| Correctness | Energies bit-identical across libs |

The backend is real (`*_r5v256` codelets; ~305× more RVV instructions than scalar). Wins evaporate when apps plan with **`FFTW_ESTIMATE`** and run many small mixed-radix transforms.

## Microbench (Orange Pi RV2)

EESSI GCC 14.3, 1 thread, `tests/bench -t 1.0`. Median MFLOPS under **MEASURE**:

| N | r5v / scalar | Speedup |
| -: | -----------: | ------: |
| 256 | 2520 / 1579 | **1.60×** |
| 1024 | 1642 / 1265 | **1.30×** |
| 4096 | 1283 / 978 | **1.31×** |
| 65536 | 797 / 752 | **1.06×** |

BPI-F3 matches within a few percent (same binaries). Pin the planner and use **≥1 s** timing — short `-t` + ESTIMATE can fake an RVV “regression.”

## End-to-end

- **QE** — `LD_PRELOAD` r5v, BLAS pinned scalar: SCF wall **~1.00×**. Stock QE uses ESTIMATE. Wisdom/MEASURE interposer recovers **~6%**, not another 3–5× ([`run-qe-fft-wisdom-ab.sh`](https://github.com/opensolvers/benchmarks/blob/main/fftw/run-qe-fft-wisdom-ab.sh)).
- **GROMACS** — isolated `PME 3D-FFT` **~1.23×**; Force still owns ~90% of the run.

Hot-codelet experiments (gather avoidance, XOR-conj, …): only XOR-conj stayed (~1–4% QE); gather rewrites lost — detail in the benchmarks repo.

## Reproduce

```bash
./build-fftw-r5v.sh
./bench-fftw-ab.sh
./run-qe-fft-ab.sh
./run-qe-fft-wisdom-ab.sh …
```

On RV2, prepend real GCC 14 bindir (compat GCC 13 otherwise wins). EasyBuild sketch: `FFTW-3.3.10-GCC-14.3.0-r5v.eb` ([hmeiland/easybuild-easyconfigs#3](https://github.com/hmeiland/easybuild-easyconfigs/pull/3)).
