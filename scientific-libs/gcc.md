---
title: GCC SpacemiT X60 mtune
description: GCC 14.3 and 15.2 EasyBuild patches for SpacemiT X60 — Orange Pi RV2 mtune A/Bs vs generic-ooo (scheduler canaries; OpenBLAS DGEMM and HPL on 15.2).
---

# GCC — SpacemiT X60 mtune

[GCC](https://gcc.gnu.org/) pipeline/tune work for the SpacemiT **X60**: EasyBuild-facing patches that teach stock GCC about `-mtune=spacemit-x60`, plus Orange Pi RV2 A/Bs that isolate **mtune only**.

| Line | Role | Source |
| ---- | ---- | ------ |
| **14.3.0** | EESSI’s current GCCcore | [benchmarks/gcc-14.3](https://github.com/opensolvers/benchmarks/tree/gcc-14.3-spacemit-x60/gcc-14.3) |
| **15.2.0** | Next foss line + OpenBLAS/HPL mtune | [benchmarks/gcc-15.2](https://github.com/opensolvers/benchmarks/tree/main/gcc-15.2) |

Upstream staging: [`spacemit-x60-gcc-tune`](https://github.com/opensolvers/spacemit-x60-gcc-tune).

> **Change one variable.** Same `-march` — only GCC **mtune** (and the patch that defines X60) differs: `spacemit-x60` vs `generic-ooo`.

> **Bottom line (RV2):** canaries move the same way on both lines (`fma_chain` / `div_mix` strongest). **14.3:** **−5.0%** / **−6.7%**. **15.2:** **−8.7%** / **−7.7%**, plus OpenBLAS DGEMM **+2.2–3.8%** and HPL N=3000 **+0.8%**. Local proof only — **not** an EESSI PR yet.

Related board notes: [Orange Pi RV2](../boards/RV2.html). BLAS / Linpack context: [BLAS](blas.html), [HPL](../apps/hpl.html).

---

## What this is

GCC builds almost everything on our EESSI RISC-V stack. Stock EESSI today is still **GCC 14.3**; a unified SpacemiT X60 patch now exists for that line as well as for **15.2**. Both add X60-aware **`-mtune` / DFA** and table-form **`xsmtvdot`**. That is separate from the [IME](../boards/RV2.html#ime-integer-matrix-extension) story (named `smt.vmadot` still needs binutils ≥ 2.46 or raw `.insn`).

---

## Patches

Same layer recipe on both lines (`type=shadd` deferred):

| Layer | Contents |
| ----- | -------- |
| A | tune/DFA (`spacemit-x60`) + table-form `xsmtvdot` / `xsmtvdotii` |
| B | 0001 → 0001b → 0002 → 0003 → 0005 (**clmul-only**) → 0004 → 0006 |

Finished RV2 semantics: **atomic@12**, **memory_cost=4**, **vector_cost** wired, **clmul@2**, **no** `type=shadd`.

| GCC | Patch | Apply |
| --- | ----- | ----- |
| 14.3.0 | [`GCC-14.3.0-spacemit-x60.patch`](https://github.com/opensolvers/benchmarks/blob/gcc-14.3-spacemit-x60/gcc-14.3/GCC-14.3.0-spacemit-x60.patch) | `patch -p1` from `gcc-14.3.0` root |
| 15.2.0 | [`GCC-15.2.0-spacemit-x60.patch`](https://github.com/opensolvers/benchmarks/blob/main/gcc-15.2/GCC-15.2.0-spacemit-x60.patch) | `patch -p1` from `gcc-15.2.0` root |

EasyBuild sketch (illustrative — not submitted):

```python
# GCCcore-14.3.0 / GCC-14.3.0
patches = ['GCC-14.3.0-spacemit-x60.patch']

# GCCcore-15.2.0 / GCC-15.2.0
patches = ['GCC-15.2.0-spacemit-x60.patch']
```

Do **not** set `EASYBUILD_OPTARCH=-mtune=spacemit-x60` until hosts actually run the patched GCCcore. Binutils IME encode stays a separate patch. Notes: [`gcc-14.3/EASYBUILD-NOTE.md`](https://github.com/opensolvers/benchmarks/blob/gcc-14.3-spacemit-x60/gcc-14.3/EASYBUILD-NOTE.md), [`gcc-15.2/EASYBUILD-NOTE.md`](https://github.com/opensolvers/benchmarks/blob/main/gcc-15.2/EASYBUILD-NOTE.md).

---

## Scheduler canaries (Orange Pi RV2)

Mean ns/call — **lower is better**. Same four kernels; 14.3 source set `rv2-gcc143-x60-ab`, 15.2 `rv2-gcc152-x60-ab-clean`.

| Kernel | 14.3 Δ% (x60 vs ooo) | 15.2 Δ% (x60 vs ooo) |
| ------ | -------------------: | -------------------: |
| `load_add_chain` | **−1.45%** | **−1.4%** |
| `fma_chain` | **−5.04%** | **−8.7%** |
| `div_mix` | **−6.73%** | **−7.7%** |
| `sh1add` | **−0.20%** | **−1.0%** |

Direction matches across versions; `fma_chain` is a bit weaker / noisier on 14.3. `sh1add` near flat — expected with `type=shadd` deferred.

Full tables: [14.3 canaries](https://github.com/opensolvers/benchmarks/tree/gcc-14.3-spacemit-x60/gcc-14.3/results/canaries), [15.2 canaries](https://github.com/opensolvers/benchmarks/tree/main/gcc-15.2/results/canaries).

---

## OpenBLAS DGEMM + HPL (15.2 only)

OpenBLAS/HPL mtune A/Bs were run on the **15.2** line only so far. Same `-march=rv64gcv_zba_zbb_zbc_zvl256b`; OpenBLAS `TARGET=RISCV64_ZVL256B`, static, OpenMP; only `-mtune` differs. DGEMM: 1 thread, core 0. HPL: N=3000, NB=192, 2×4 — host built with EESSI; linked OpenBLAS is the A/B axis.

| Axis | Δ% (x60 vs ooo) |
| ---- | --------------: |
| DGEMM N=512 / 1024 / 2048 | **+2.2% / +2.3% / +3.8%** GF/s |
| HPL N=3000 | **+0.8%** Gflops (both PASSED) |

Checksums matched across mtune for all DGEMM sizes. Summaries: [benchmarks `gcc-15.2/results/openblas-hpl/`](https://github.com/opensolvers/benchmarks/tree/main/gcc-15.2/results/openblas-hpl).

Absolute single-thread DGEMM (~2.2–2.4 GF/s) is modest for ZVL256B; the point of the run is **mtune isolation**, not peak system HPL.

---

## Reading

X60-aware scheduling moves micro kernels that care about FMA / divide mix on both **14.3** (today’s EESSI GCCcore) and **15.2**. The 15.2 line also shows a few percent in OpenBLAS DGEMM and a sub-percent HPL sanity run. Useful for a current or future EESSI `foss` bump — not a substitute for the OpenBLAS `gemv_n` correctness fix or hand RVV app kernels.

**Measured:** 2026-08 on Orange Pi RV2 (SpaceMiT X60).
