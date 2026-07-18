# Voiceover script — U74 on VisionFive 2 (English)

**Target duration:** 72–78 seconds  
**Pace:** ~140 words/minute · **Word count:** ~175

Record as one take per section if easier; align to [`storyboard.md`](storyboard.md).

---

## [0:00 – 0:14] Setup — SLIDE 1 → 2

OpenSolvers benchmarks open-source scientific software on real RISC-V hardware — through the EESSI stack, without rebuilding every application.

Our first board is the StarFive VisionFive 2: four SiFive U74 cores at one point five gigahertz. Scalar RV64GC — no vector extension. On this chip, our optimisation target is the **Scalar** path: tuned BLAS kernels, not RVV.

---

## [0:14 – 0:28] Problem — SLIDE 3

Out of the box, stock OpenBLAS on EESSI uses a generic C GEMM kernel.

High Performance Linpack on four cores reaches about **three point one three** gigaflops per second. The answer is correct — but we’re leaving throughput on the table.

---

## [0:28 – 0:52] Fix — SLIDE 4 + TERMINAL CLIP

We built an U74-tuned OpenBLAS with a hand-tuned **four-by-four DGEMM** micro-kernel — `TARGET=U74` — and swapped it in at runtime with FlexiBLAS.

Same HPL binary. No rebuild.

*(Terminal clip: module load, FlexiBLAS list, optional xhpl banner)*

Single-core DGEMM goes from about **one point four** to **one point eight** gigaflops per second. Four-core GEMM hits **six point three**. The kernel change propagates straight into Linpack.

---

## [0:52 – 1:15] Result — SLIDE 5 → 6

End-to-end HPL on the VisionFive 2: **three point one three** to **five point two eight** gigaflops per second — **one point seven times** faster. Wall time drops from **two hundred thirteen** seconds to **one hundred twenty-six**.

Same methodology we use on SpaceMiT X60 boards — change one backend, verify correctness first — but on U74 the story is **silicon-tuned scalar**, not fixing a broken vector path.

Everything is documented at **opensolvers dot com**.

---

## Pronunciation / on-screen numbers

| Spoken | On screen |
| ------ | --------- |
| three point one three | 3.13 GFLOP/s |
| five point two eight | 5.28 GFLOP/s |
| one point seven times | 1.69× |
| one point four → one point eight | 1.4 → 1.77 GFLOP/s |
| six point three | 6.31 GFLOP/s |
| two hundred thirteen → one hundred twenty-six | 213 s → 126 s |
