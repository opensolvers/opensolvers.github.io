---
title: GCC 15.2 SpacemiT X60 mtune
description: GCC 15.2.0 EasyBuild patch for SpacemiT X60 — Orange Pi RV2 mtune A/Bs vs generic-ooo (scheduler canaries, OpenBLAS DGEMM, HPL).
---

# GCC 15.2 — SpacemiT X60 mtune

[GCC](https://gcc.gnu.org/) **15.2.0** pipeline/tune work for the SpacemiT **X60**: a single EasyBuild-facing patch that teaches stock GCC about `-mtune=spacemit-x60`, plus Orange Pi RV2 A/Bs that isolate **mtune only**.

Benchmark source: [opensolvers/benchmarks/gcc-15.2](https://github.com/opensolvers/benchmarks/tree/gcc-15.2-spacemit-x60/gcc-15.2). Upstream staging: [`spacemit-x60-gcc-tune`](https://github.com/opensolvers/spacemit-x60-gcc-tune).

> **Change one variable.** Same `-march`, same OpenBLAS target / HPL config — only GCC **mtune** (and the patch that defines X60) differs: `spacemit-x60` vs `generic-ooo`.

> **Bottom line (RV2):** scheduler canaries **−8.7%** / **−7.7%** ns/call on `fma_chain` / `div_mix`; OpenBLAS DGEMM **+2.2–3.8%** GF/s; modest HPL N=3000 **+0.8%**. Local proof only — **not** an EESSI PR yet.

Related board notes: [Orange Pi RV2](../boards/RV2.html). BLAS / Linpack context: [BLAS](blas.html), [HPL](../apps/hpl.html).

---

## What this is

GCC is the compiler that builds almost everything on our EESSI RISC-V stack. Stock EESSI today is still **GCC 14.3**; X60-aware **`-mtune` / DFA** and table-form **`xsmtvdot`** land in this **15.2.0** patch set. That is separate from the [IME](../boards/RV2.html#ime-integer-matrix-extension) story (named `smt.vmadot` still needs binutils ≥ 2.46 or raw `.insn`).

---

## Patch

[`GCC-15.2.0-spacemit-x60.patch`](https://github.com/opensolvers/benchmarks/blob/gcc-15.2-spacemit-x60/gcc-15.2/GCC-15.2.0-spacemit-x60.patch) — one file for stock GCC 15.2.0 (`patch -p1`).

| Layer | Contents |
| ----- | -------- |
| A | tune/DFA (`spacemit-x60`) + table-form `xsmtvdot` / `xsmtvdotii` |
| B | 0001 → 0001b → 0002 → 0003 → 0005 (**clmul-only**) → 0004 → 0006 |

Finished RV2 semantics: **atomic@12**, **memory_cost=4**, **vector_cost** wired, **clmul@2**, **no** `type=shadd` (deferred).

```bash
# From extracted gcc-15.2.0 source root (EasyBuild via patches = [...]):
patch -p1 < GCC-15.2.0-spacemit-x60.patch
```

EasyBuild sketch (illustrative — not submitted):

```python
patches = [
    'GCC-15.2.0-spacemit-x60.patch',
]
```

Do **not** set `EASYBUILD_OPTARCH=-mtune=spacemit-x60` until hosts actually run this patched GCCcore. Binutils IME encode stays a separate patch. Details: [`EASYBUILD-NOTE.md`](https://github.com/opensolvers/benchmarks/blob/gcc-15.2-spacemit-x60/gcc-15.2/EASYBUILD-NOTE.md).

---

## Scheduler canaries (Orange Pi RV2)

Source set: `rv2-gcc152-x60-ab-clean`. Mean ns/call — **lower is better**.

| Kernel | Δ% x60 vs `generic-ooo` |
| ------ | ----------------------: |
| `load_add_chain` | **−1.4%** |
| `fma_chain` | **−8.7%** |
| `div_mix` | **−7.7%** |
| `sh1add` | **−1.0%** |

Full tables: [benchmarks `results/canaries/`](https://github.com/opensolvers/benchmarks/tree/gcc-15.2-spacemit-x60/gcc-15.2/results/canaries).

---

## OpenBLAS DGEMM + HPL

Same `-march=rv64gcv_zba_zbb_zbc_zvl256b`; OpenBLAS `TARGET=RISCV64_ZVL256B`, static, OpenMP; only `-mtune` differs. DGEMM: 1 thread, core 0. HPL: N=3000, NB=192, 2×4 — host built with EESSI; linked OpenBLAS is the A/B axis.

| Axis | Δ% (x60 vs ooo) |
| ---- | --------------: |
| DGEMM N=512 / 1024 / 2048 | **+2.2% / +2.3% / +3.8%** GF/s |
| HPL N=3000 | **+0.8%** Gflops (both PASSED) |

Checksums matched across mtune for all DGEMM sizes. Summaries: [benchmarks `results/openblas-hpl/`](https://github.com/opensolvers/benchmarks/tree/gcc-15.2-spacemit-x60/gcc-15.2/results/openblas-hpl).

Absolute single-thread DGEMM (~2.2–2.4 GF/s) is modest for ZVL256B; the point of the run is **mtune isolation**, not peak system HPL.

---

## Reading

X60-aware scheduling moves micro kernels that care about FMA / divide mix; the same tune shows up as a few percent in OpenBLAS DGEMM and a sub-percent HPL sanity run. Useful for a future EESSI `foss` bump — not a substitute for the OpenBLAS `gemv_n` correctness fix or hand RVV app kernels.

**Measured:** 2026-08 on Orange Pi RV2 (SpaceMiT X60).
