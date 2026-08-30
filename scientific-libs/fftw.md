# FFTW

**Video:** [When RVV FFT Wins 1.6× — and Gives 0% in Quantum ESPRESSO](https://www.youtube.com/watch?v=FlumrCEUIBE) — [all videos](../videos.html)

[FFTW](https://www.fftw.org/) 3.3.10 with the **RISC-V Vector (`r5v`) SIMD backend** — a clean A/B against a scalar build of the *same source* with identical compiler and flags. The only variable is `--enable-r5v` (from [rdolbeau's `r5v-test-release-005`](https://github.com/rdolbeau)).

Benchmark source: [opensolvers/benchmarks/fftw](https://github.com/opensolvers/benchmarks/tree/main/fftw) — build/bench scripts, QE wisdom A/B (`run-qe-fft-wisdom-ab.sh`), and `r5v` simd patches.

Relevant to [Quantum ESPRESSO](../apps/qe.html) and [GROMACS](../apps/gromacs.html): real apps spend large fractions on FFT. A FlexiBLAS swap does not touch FFT — swap the library via `LD_PRELOAD` instead. Also: [3.31× GROMACS Force backend](https://www.youtube.com/watch?v=COayFhBa0as) (why FFT micro wins can dilute).

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

## Cross-board — Banana Pi BPI-F3

Same `tests/bench` A/B and the **same** r5v/scalar `libfftw3.so.3.6.10` binaries built on the RV2, on [Banana Pi F3](../boards/F3.html):

| size | estimate r5v / scal | **MEASURE r5v / scal** | **r5v speedup (MEASURE)** |
| ---: | ------------------: | ---------------------: | ------------------------: |
| 256 | 2196 / 1386 | **2518 / 1576** | **1.60×** |
| 1024 | 720 / 746 | **1634 / 1273** | **1.28×** |
| 4096 | 302 / 361 | **1276 / 1033** | **1.24×** |
| 16384 | 424 / 330 | **943 / 817** | **1.15×** |
| 65536 | 178 / 169 | 723 / 741 | **0.98×** |
| 262144 | 203 / 160 | **729 / 685** | **1.06×** |

Matches the RV2 within a few percent at every cache-resident size (1.60× @ 256 identical). At N≥64K both boards are bandwidth-bound; the F3's 0.98× at N=65536 is within run-to-run noise of the RV2's 1.06×.

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

- **QE plans with `FFTW_ESTIMATE`, not `MEASURE`.** The RVV advantage above is largely a planner effect; under `estimate` the two libs are near-parity — and `estimate` is what QE uses by default.
- **Thousands of small mixed-radix transforms**, not the cache-resident power-of-two sizes where RVV shines.

Contrast [GROMACS](../apps/gromacs.html): swapping `libfftw3f` wins **1.23×** on the isolated `PME 3D-FFT` step (but `Force` = 90% of the run).

## Wisdom / MEASURE in QE — ~6%, not another 3–5×

Hypothesis: force `FFTW_MEASURE` (or cached wisdom) without patching QE, and the microbench planner gap should appear in SCF. Scripts: `fftw-est2meas-interposer.c` (`LD_PRELOAD` remaps ESTIMATE→MEASURE) + `run-qe-fft-wisdom-ab.sh`.

### 1-D microbench still shows 3–5×

Cold ESTIMATE vs MEASURE on power-of-two complex DFTs (r5v lib): e.g. N=4096 **~294 → ~1285 MFLOPS**. After importing MEASURE wisdom, `FFTW_ESTIMATE` matches MEASURE throughput with ~ms plan cost.

### Serial QE 64-atom (`si-super-64.in`, r5v FFTW)

Energy bit-identical (`−506.67980304 Ry`). WALL seconds:

| step | `fftw` | `init_run` | `PWSCF` |
| ---- | -----: | ---------: | -----: |
| 1 ESTIMATE (stock QE) | 85.03 | 28.45 | 188.91 |
| 2 MEASURE collect | 83.38 | 28.20 | 187.76 |
| 3 ESTIMATE + MEASURE wisdom | 81.76 | 20.20 | 180.49 |
| 4 MEASURE + wisdom | 80.08 | 19.91 | 178.63 |

Best vs stock: **`fftw` / `PWSCF` ≈ 1.06×** (~10 s wall). PATIENT wisdom is within noise of MEASURE. The planner trap is real in isolation; QE's ~2362 many-DFTs barely move past MEASURE.

### MPI (`NP=4` / `NP=8`, overlay QE 7.5)

Same cell; r5v via `LD_PRELOAD` + `RTLD_DEEPBIND` (binary RPATH pins stock FFTW.MPI). Per-rank wisdom merged with `merge-fftw-wisdom`.

| NP | best `fftw` vs ESTIMATE | best `PWSCF` |
| -: | ----------------------: | -----------: |
| 4 | ~3% | ~2% |
| 8 | ~3% (`16.06` → `15.53` s) | ~6% (`44.93` → `42.21` s) |

MPI already cuts wall ~3× vs serial; planner quality is not the remaining lever.

## Hot codelets — XOR-conj helps a little; gather rewrites do not

QE MPI wisdom is mostly **solvers**; only ~11 entries are `*_r5v256` codelets (`t2bv_8`, `t2fv_*`, `n2fv_16`, …). `t2bv_8` @ r5v256 has **10× `vrgather`** (index already CSE’d) — gather latency on K1, as Dolbeau noted in [FFTW#371](https://github.com/FFTW/fftw3/issues/371).

| Experiment | Result |
| ---------- | ------ |
| Store-shuffle instead of `vrgather` (`VDUPL`/`FLIP_RI`/…) | **0.77–0.89×** — reverted |
| XOR `VCONJ` + fused `VBYI` (kept) | **+1–7%** micro; QE NP=8 ESTIMATE **`fftw` 1.02×**, **`PWSCF` 1.04×** |
| NEON-style `VFMAI` / `VZMUL` | within noise / slight `many` loss — reverted |
| Slide/store `FLIP_RI` alone | gather still faster (~6.9 vs ~11–12 ns) |
| Hand-split `t2bv_8` (`vlseg2`/`vsseg2`) | bit-exact, **~0.48×** vs stock — reverted |

On X60, short `vrgather` beats the gather-avoidance tricks we tried. Further QE wall gains need a non-FFT hotspot or a larger algorithmic change.

## EasyBuild module

`FFTW-3.3.10-GCC-14.3.0-r5v.eb` packages the r5v build as a reproducible EESSI module (`--enable-r5v`, pinned `-march=rv64imafdcv_zvl256b`). Experimental — tracked on [hmeiland/easybuild-easyconfigs#3](https://github.com/hmeiland/easybuild-easyconfigs/pull/3), not yet upstream-ready.

## Reproduce

```bash
./build-fftw-r5v.sh              # needs $HOME/fftw-r5v.tar.gz on the board
./bench-fftw-ab.sh               # writes fftw-proper.log
./run-qe-fft-ab.sh               # QE FFT-axis A/B (LD_PRELOAD)
./run-qe-fft-wisdom-ab.sh …      # ESTIMATE → MEASURE → wisdom steps
NP=8 ./run-qe-fft-wisdom-ab.sh … # MPI + merge-fftw-wisdom
```

**Gotcha (Orange Pi RV2):** `module load GCCcore/14.3.0` does not repath `gcc` — EESSI's compat GCC 13.4.0 keeps winning. Prepend the real GCC 14 bindir explicitly (see `build-fftw-r5v.sh`). On [Banana Pi F3](../boards/F3.html) the plain `module load GCC/14.3.0` repaths correctly.
