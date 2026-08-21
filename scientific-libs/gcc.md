---
title: GCC SpacemiT X60 mtune
description: GCC 14.3 and 15.2 EasyBuild patches for SpacemiT X60 — Orange Pi RV2 mtune A/Bs vs generic-ooo (canaries; OpenBLAS DGEMM and HPL on both lines).
---

# GCC — SpacemiT X60 mtune

**Video:** [GCC 15.2 on RISC-V: Teaching mtune=spacemit-x60 (−8.7% Canaries, +3.8% DGEMM)](https://www.youtube.com/watch?v=guKeNzHXm70) — [all videos](../videos.html)

[GCC](https://gcc.gnu.org/) pipeline/tune work for the SpacemiT **X60**: EasyBuild-facing patches that teach stock GCC about `-mtune=spacemit-x60`, plus Orange Pi RV2 A/Bs that isolate **mtune only**.

| Line | Role | Source |
| ---- | ---- | ------ |
| **14.3.0** | EESSI’s current GCCcore | [benchmarks/gcc-14.3](https://github.com/opensolvers/benchmarks/tree/main/gcc-14.3) |
| **15.2.0** | Next foss line | [benchmarks/gcc-15.2](https://github.com/opensolvers/benchmarks/tree/main/gcc-15.2) |

Upstream staging: [`spacemit-x60-gcc-tune`](https://github.com/opensolvers/spacemit-x60-gcc-tune).

> **Change one variable.** Same `-march` — only GCC **mtune** (and the patch that defines X60) differs: `spacemit-x60` vs `generic-ooo`.

> **Bottom line (RV2):** canaries move both ways on `fma_chain` / `div_mix`. **14.3:** **−5.0%** / **−6.7%** ns/call; DGEMM **−3…−7%**; HPL **+6.8%**. **15.2:** **−8.7%** / **−7.7%**; DGEMM **+2–4%**; HPL **+0.8%**. Local proof only — **not** an EESSI PR yet.

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
| 14.3.0 | [`GCC-14.3.0-spacemit-x60.patch`](https://github.com/opensolvers/benchmarks/blob/main/gcc-14.3/GCC-14.3.0-spacemit-x60.patch) | `patch -p1` from `gcc-14.3.0` root |
| 15.2.0 | [`GCC-15.2.0-spacemit-x60.patch`](https://github.com/opensolvers/benchmarks/blob/main/gcc-15.2/GCC-15.2.0-spacemit-x60.patch) | `patch -p1` from `gcc-15.2.0` root |

```python
patches = ['GCC-14.3.0-spacemit-x60.patch']  # or GCC-15.2.0-…
```

Do **not** set `EASYBUILD_OPTARCH=-mtune=spacemit-x60` until hosts actually run the patched GCCcore. Binutils IME encode stays a separate patch.

---

## Scheduler canaries (Orange Pi RV2)

Mean ns/call — **lower is better**.

| Kernel | 14.3 Δ% (x60 vs ooo) | 15.2 Δ% (x60 vs ooo) |
| ------ | -------------------: | -------------------: |
| `load_add_chain` | **−1.45%** | **−1.4%** |
| `fma_chain` | **−5.04%** | **−8.7%** |
| `div_mix` | **−6.73%** | **−7.7%** |
| `sh1add` | **−0.20%** | **−1.0%** |

---

## OpenBLAS DGEMM + HPL

Same `-march=rv64gcv_zba_zbb_zbc_zvl256b`; OpenBLAS `TARGET=RISCV64_ZVL256B`, static; only `-mtune` differs. HPL N=3000, NB=192, 2×4 — both PASSED.

| Axis | 14.3 Δ% (x60 vs ooo) | 15.2 Δ% (x60 vs ooo) |
| ---- | -------------------: | -------------------: |
| DGEMM N=512 | **−6.9%** | **+2.2%** |
| DGEMM N=1024 | **−6.7%** | **+2.3%** |
| DGEMM N=2048 | **−3.0%** | **+3.8%** |
| HPL N=3000 | **+6.8%** | **+0.8%** |

**Sign flip on DGEMM:** 14.3 x60 mtune *slows* single-thread DGEMM vs `generic-ooo` on this run; 15.2 speeds it up. HPL still nudges positive on both. Summaries: [gcc-14.3/results/openblas-hpl](https://github.com/opensolvers/benchmarks/tree/main/gcc-14.3/results/openblas-hpl), [gcc-15.2/results/openblas-hpl](https://github.com/opensolvers/benchmarks/tree/main/gcc-15.2/results/openblas-hpl).

---

## Reading

X60-aware scheduling moves micro kernels that care about FMA / divide mix on both lines. End-to-end DGEMM/HPL deltas are small and **version-dependent** — useful for a future EESSI `foss` bump, not a substitute for the OpenBLAS `gemv_n` correctness fix or hand RVV app kernels.

**Measured:** 2026-08 on Orange Pi RV2 (SpaceMiT X60).
