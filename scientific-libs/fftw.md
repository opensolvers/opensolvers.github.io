# FFTW

[FFTW](https://www.fftw.org/) 3.3.10 with the **RISC-V Vector (`r5v`) SIMD backend** — a clean A/B against a scalar build of the *same source* with identical compiler and flags. The only variable is `--enable-r5v` (from [rdolbeau's `r5v-test-release-005`](https://github.com/rdolbeau)).

Benchmark source: [opensolvers/benchmarks/fftw](https://github.com/opensolvers/benchmarks/tree/main/fftw) — `build-fftw-r5v.sh` and `bench-fftw-ab.sh`.

Relevant to [Quantum ESPRESSO](../apps/qe.html) and [GROMACS](../apps/gromacs.html): real apps spend large fractions on FFT. A FlexiBLAS swap does not touch FFT — swap the library via `LD_PRELOAD` instead (see `run-qe-fft-ab.sh` in the benchmarks repo).

## Orange Pi RV2 (SpaceMiT X60, 1 thread)

EESSI `2025.06-001`, GCC 14.3.0, `-O3 -march=rv64imafdcv_zvl256b`. 1D complex-to-complex via FFTW's own `tests/bench`, `-t 1.0`.

### The RVV backend is real

| Build | Library size | RVV instr count | Codelets in plan |
| ----- | -----------: | --------------: | ---------------- |
| **r5v** (`--enable-r5v`) | 11 MB | **224,354** | `n1fv_16_r5v256`, `t3fv_4_r5v256`, … |
| **scalar** (control) | 924 KB | 734 | (none) |

The r5v library emits **~305×** more vector instructions than scalar, and FFTW's planner actually selects `*_r5v256` codelets (confirmed via `bench -v2`).

### Performance — `FFTW_MEASURE` default

Median MFLOPS; higher = faster. Under the **`FFTW_MEASURE`** planner (the default when no `-o` is passed), r5v beats scalar at every size:

| size | estimate r5v / scalar | **MEASURE r5v / scalar** | **r5v speedup** |
| ---: | --------------------: | -----------------------: | --------------: |
| 256 | 2228 / 1388 | **2520 / 1579** | **1.60×** |
| 1024 | 717 / 747 | **1642 / 1265** | **1.30×** |
| 4096 | 303 / 360 | **1283 / 978** | **1.31×** |
| 16384 | 381 / 276 | **964 / 797** | **1.21×** |
| 65536 | 148 / 142 | **797 / 752** | **1.06×** |
| 262144 | 171 / 138 | **717 / 664** | **1.08×** |

Largest gain on cache-resident transforms (**1.60×** @ N=256); tapers to **~1.06×** as transforms become memory-bandwidth-bound (≥64K).

## Planner choice matters more than codelets

The biggest lever on this hardware is **planner choice** — worth **3–5×**, independent of RVV. `FFTW_ESTIMATE` grossly under-plans large transforms:

| size | estimate → MEASURE (r5v) | gain |
| ---: | -----------------------: | ---: |
| 4096 | 303 → 1283 | **4.2×** |
| 16384 | 381 → 964 | **2.5×** |
| 65536 | 148 → 797 | **5.4×** |
| 262144 | 171 → 717 | **4.2×** |

`patient` was within noise of `MEASURE` where it completed, but planning time blows up at large N (>35 min at N=262144). **`FFTW_MEASURE`** (or cached wisdom) is the sweet spot — never `FFTW_ESTIMATE` on the X60/K1.

### Methodology trap

A first pass with `estimate` and `-t 0.3` appeared to show a **2× RVV regression** at N=262144. With honest timing, r5v is faster under both planners (717 vs 664 MFLOPS under MEASURE). Pin the planner and use **≥1 s** timing before trusting any single FFT A/B point.

## End-to-end — Quantum ESPRESSO gets ~0% from RVV FFTW

The microbench shows real RVV codelets (**1.06–1.60×**). Does that survive into a real DFT run? [`run-qe-fft-ab.sh`](https://github.com/opensolvers/benchmarks/blob/main/fftw/run-qe-fft-ab.sh) swaps **only** the FFT library (`LD_PRELOAD`) on serial `pw.x`, with BLAS pinned to scalar OpenBLAS via FlexiBLAS.

**Correctness:** total energy bit-identical across backends (2-atom **−14.57861334 Ry**; 64-atom **−506.67991945 Ry**).

64-atom SCF, WALL seconds ([Orange Pi RV2](../boards/RV2.html)):

| Timer | Scalar FFTW | r5v (RVV) FFTW | r5v speedup |
| ----- | ----------: | -------------: | ----------: |
| `fftw` | 112.24 | 110.09 | **1.019×** |
| `vloc_psi` | 105.29 | 103.08 | 1.021× |
| **`PWSCF` (total)** | **248.49** | **248.10** | **1.002×** |

**~0.2% wall, ~1.9% inside `fftw`** — even though `fftw` is ~45% of runtime. Why the 1.6× micro-win evaporates:

- **QE plans with `FFTW_ESTIMATE`, not `MEASURE`.** The RVV advantage in section (2) is largely a planner effect; under `estimate` the two libs are near-parity — and `estimate` is what QE uses (it cannot afford `MEASURE` across thousands of transient transforms).
- **3260 small mixed-radix transforms**, not the cache-resident power-of-two sizes where RVV shines.

On the X60, **neither the BLAS axis nor the FFT axis moves a real QE SCF** with today's drop-in vectorized libraries. Contrast [GROMACS](../apps/gromacs.html): swapping `libfftw3f` wins **1.23×** on the isolated `PME 3D-FFT` step (but `Force` = 90% of the run).

## EasyBuild module

`FFTW-3.3.10-GCC-14.3.0-r5v.eb` packages the r5v build as a reproducible EESSI module (`--enable-r5v`, pinned `-march=rv64imafdcv_zvl256b`). Experimental — tracked on [hmeiland/easybuild-easyconfigs#3](https://github.com/hmeiland/easybuild-easyconfigs/pull/3), not yet upstream-ready.

## Reproduce

```bash
./build-fftw-r5v.sh    # needs $HOME/fftw-r5v.tar.gz on the board
./bench-fftw-ab.sh     # writes fftw-proper.log
./run-qe-fft-ab.sh     # QE FFT-axis A/B (LD_PRELOAD)
```

**Gotcha (Orange Pi RV2):** `module load GCCcore/14.3.0` does not repath `gcc` — EESSI's compat GCC 13.4.0 keeps winning. Prepend the real GCC 14 bindir explicitly (see `build-fftw-r5v.sh`). On [Banana Pi F3](../boards/F3.html) the plain `module load GCC/14.3.0` repaths correctly.
