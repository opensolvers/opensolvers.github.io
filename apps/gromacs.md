# GROMACS

[GROMACS](https://www.gromacs.org/) `mdrun` PME molecular dynamics on the SpaceMiT X60 — two independent backend axes measured end-to-end:

1. **FFT axis** — swap single-precision `libfftw3f.so.3` via `LD_PRELOAD` (stock EESSI build, `SIMD: None`)
2. **Force axis** — hand-written **`impl_riscv_rvv/`** SIMD backend (`SIMD: RISCV_RVV`) or upstream 2026.3 autovec 1×1 kernel

Benchmark source: [opensolvers/benchmarks/gromacs](https://github.com/opensolvers/benchmarks/tree/main/gromacs) — `gen-water-box.sh`, `run-gmx-fft-ab.sh`, and [`rvv-backend/`](https://github.com/opensolvers/benchmarks/tree/main/gromacs/rvv-backend) (patch + `apply-and-build.sh`).

FFT microbench numbers: [FFTW](../scientific-libs/fftw.html). BLAS-axis counterpart: [Quantum ESPRESSO](qe.html).

---

## FFT axis — PME 3D-FFT (`run-gmx-fft-ab.sh`)

Swaps only `libfftw3f` under one unchanged EESSI `GROMACS/2026.2-foss-2025b` binary. BLAS pinned to scalar OpenBLAS; serial (`-ntmpi 1 -ntomp 1`) by design to isolate FFT.

On the stock build GROMACS reports **`SIMD instructions: None`** — force kernels are scalar, so `PME 3D-FFT` / `PME mesh` are the swapped rows; `Force` (90% of runtime) is the built-in control.

### Correctness — RVV FFT reproduces scalar MD

Same `mdrun`, same `md.tpr`, 17.2k-atom SPC water, 2000 steps:

| FFT backend | Avg potential energy | Δ vs scalar |
| ----------- | -------------------: | ----------: |
| Scalar (`libfftw3f`, ~944 KB) | **−236755 kJ/mol** | — |
| r5v / RVV (`libfftw3f`, ~14 MB, 282k RVV instr) | **−236592 kJ/mol** | **0.069%** |

### Performance — isolated FFT step (Orange Pi RV2)

Serial, 48×48×48 PME grid. GROMACS cycle accounting (WALL):

| Activity | Scalar | r5v (RVV) | Speedup | Kind |
| -------- | -----: | --------: | ------: | ---- |
| **`PME 3D-FFT`** | 21.095 s | 17.152 s | **1.23×** | **FFT (swapped axis)** |
| `PME mesh` (total) | 73.281 s | 69.792 s | 1.05× | FFT + spread/gather/solve |
| **`Force`** | 821.35 s | 819.97 s | 1.00× | scalar control (90%) |
| `Neighbor search` | 6.731 s | 6.714 s | 1.00× | scalar control |

RVV wins **~1.23×** on the isolated 3D-FFT step; **`Force` confirms the A/B** (821.35 vs 819.97 s, 0.17% noise). Whole-app effect is small because `Force` dominates.

---

## Force axis — RVV SIMD backend (`rvv-backend/`)

The scalar `Force` term that dilutes the FFT swap is itself optimizable. [`rvv-backend/`](https://github.com/opensolvers/benchmarks/tree/main/gromacs/rvv-backend) ships a hand-written **`impl_riscv_rvv/`** backend for GROMACS **2026.2** (~1545 lines, float/mixed precision) — ARM SVE as the VLA template.

Built on the X60 under EESSI GCC 14.3 with `GMX_SIMD=RISCV_RVV`. `gmx --version` reports **`SIMD instructions: RISCV_RVV`**; float SIMD unit tests **158/158** pass.

Same serial protocol (SPC water PME, `md.tpr`, 2000 steps, `-ntmpi 1 -ntomp 1 -nb cpu -pme cpu`):

| Metric | Scalar (`SIMD: None`) | RVV (`SIMD: RISCV_RVV`) | Speedup |
| ------ | --------------------: | ------------------------: | ------: |
| **`Force` wall** | 821.35 s | **187.37 s** | **4.38×** |
| Total wall | 910.33 s | **275.40 s** | **3.31×** |
| Performance | 0.380 ns/day | **1.256 ns/day** | **3.31×** |
| Avg potential energy | −236755 kJ/mol | −236653 kJ/mol | Δ 0.043% (correct) |

**The profile inverts:** with `Force` cut ~4.4×, PME mesh becomes the leading cost (28% vs 8% of runtime) — so the FFT axis matters more after vectorizing `Force`.

Status: captured as a patch for reproduction; **no upstream PR yet**. Double precision not ported.

---

## Force axis — three levers compared

All measured on Orange Pi RV2, same SPC water system, 2000 steps, 1 core (unless noted):

| Force lever | GROMACS | Mechanism | `Force` wall | ns/day | vs scalar |
| ----------- | ------- | --------- | -----------: | -----: | --------: |
| Scalar 4×4 (baseline) | 2026.2 | none (`SIMD: None`) | 821.35 s | 0.380 | 1.00× |
| GCC `-march=…v` 4×4 autovec | 2026.2 | compiler on 4×4 kernel | ~836 s* | 0.372 | **0.98× (slower)** |
| **Upstream 1×1 autovec** | **2026.3** | clang `#pragma`, `GMX_NBNXN_PLAINC_1X1=1` | **480.14 s** | **0.543** | **1.43×** |
| **Hand-written `impl_riscv_rvv/`** | **2026.2** | `GMX_SIMD=RISCV_RVV` | **187.37 s** | **1.256** | **3.31×** |

<sub>*4×4 autovec Force scaled from 1000-step run; ns/day directly comparable.</sub>

The **2026.3 autovec 1×1** path (clang ≥ 19, `GMX_ENABLE_NBNXM_CPU_VECTORIZATION=on`) is a real win — **1.74× whole-app** vs the same binary's default 4×4 — but the hand-written backend is still **2.31× faster** whole-app (1.256 vs 0.543 ns/day). Autovec is VLEN-agnostic and upstream-native; the dedicated backend targets fixed VLEN=256 (`-mrvv-vector-bits=zvl`).

**Free lever:** threading — serial A/B wastes 7 of 8 cores. Same system at `-ntomp 8` reaches **~2.60 ns/day** (~6.9× vs serial baseline).

---

## Backend-dilution spectrum

| Probe | Axis | Isolated speedup | Whole-app effect |
| ----- | ---- | ---------------: | ---------------- |
| [OpenBLAS verification](../scientific-libs/blas.html#verification) | BLAS | ~2.3× | (pure kernel) |
| [BLIS](../scientific-libs/blis.html) vs OpenBLAS | BLAS | ~1.29× (1T, N=4096) | (microbench) |
| [QE](qe.html) (DFT SCF) | BLAS | ~1.5–2.0× on BLAS routines | ~1.2–1.3× (FFT untouched) |
| [QE + FFTW](../scientific-libs/fftw.html) | FFT | ~1.02× on `fftw` | **~0%** (`FFTW_ESTIMATE`) |
| **GROMACS FFT** (this page) | FFT | **~1.23× on `PME 3D-FFT`** | small (`Force` = 90%) |
| **GROMACS Force** (`rvv-backend/`) | **SIMD** | **~4.38× on `Force`** | **~3.31×** |

GROMACS FFT is the mirror of QE: QE speeds up BLAS and is diluted by FFT; the FFT swap is diluted by scalar `Force`. Vectorizing `Force` moves the whole app — then PME/FFT becomes the next target.

---

## Gotchas

- GROMACS mixed precision links **`libfftw3f.so.3`** (single precision), not double-precision `libfftw3.so.3` from QE/FFTW work.
- Fixed `-nsteps` keeps FFT call count identical between backends (4002 here).
- RVV `libfftw3f` build is slow to compile (~2 h on one X60); scalar ~15 min.
- GCC `-march=rv64gcv_zvl256b` on stock 4×4 `Force` emits 164k RVV instrs but runs **~2% slower** — gather-heavy kernel defeats autovec.
- 2026.3 autovec 1×1 needs **clang ≥ 19** and runtime **`GMX_NBNXN_PLAINC_1X1=1`**; confirm via `Using plain-C-1x1` in `md.log`.
- RVV-backend builds need GCC 14.3's `libstdc++` (`CXXABI_1.3.15`) on `LD_LIBRARY_PATH` at runtime.

## Reproduce

```bash
# FFT axis
module load GROMACS/2026.2-foss-2025b
./gen-water-box.sh
bash run-gmx-fft-ab.sh v1

# Force axis (hand-written RVV backend)
cd gromacs/rvv-backend
./apply-and-build.sh /path/to/gromacs-2026.2
```

**Toolchain:** GROMACS 2026.2 / foss-2025b (EESSI), FFTW 3.3.10 `libfftw3f`, Orange Pi RV2 (SpaceMiT X60).
