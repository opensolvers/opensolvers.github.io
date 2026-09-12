---
title: OpenSolvers — RISC-V scientific software benchmarks
description: Benchmark notes for open-source scientific libraries, applications, and AI inference engines on consumer RISC-V boards — HPL, BLAS, Quantum ESPRESSO, llama.cpp, GROMACS, LAMMPS, OpenFOAM, waLBerla, PETSc, EESSI, and FlexiBLAS.
permalink: /
---

OpenSolvers explores how open-source scientific software runs on real hardware — starting with **RISC-V** boards and the tools that make that practical (EESSI, OpenBLAS, and friends). We also tune **AI inference** ([llama.cpp](apps/llamacpp.html), [ONNX Runtime](apps/onnx.html)) on the same cores (RVV, IME). If the work is useful, [sponsor OpenSolvers on GitHub](https://github.com/sponsors/opensolvers).

## Why single-board computers?

The improvements we chase live at the **core** of the stack — BLAS kernels, vector backends, ISA bugs, correctness that only shows up on real silicon. An SBC gives that microarchitecture with a feedback loop in hours, not queue days. Get the kernels right on a board, and the same modules **scale** to a cluster.

## Highlights

- **OpenBLAS `gemv_n`** — stock RVV failed HPL / ELPA / QE with `nan`; patched EESSI restores correctness ([HPL](apps/hpl.html), [EESSI blog](https://www.eessi.io/docs/blog/2026/07/12/risc-v-x60-openblas-hpl/))
- **IME on X60** — ~42–48 GOP/s full GEMM, **68** with TCM when B fits; **TCM off** for LLM e2e ([RV2](boards/RV2.html#best-ime-results-step-2-panel--memory-2026-09-02))
- **ONNX / llama.cpp** — real decode via CompInt8 IME; Q4_0 IME wins prefill, RVV wins token-gen; Q8_0 hybrid restores decode
- **GROMACS** — hand RVV `Force` **3.31×** whole-app; FFT micro wins alone do not move the needle
- **FFTW / QE** — RVV FFT **1.06–1.60×** in isolation, **~0%** drop-in under `FFTW_ESTIMATE`

[Videos](videos.html) · [YouTube](https://www.youtube.com/@opensolvers)

## What we optimise on the board

![Scalar, Vector, Custom, and GPU compute paths on the board](assets/images/compute-backends.svg)

| Path | On our boards |
| ---- | ------------- |
| **Scalar** | Correctness baseline (`rv64gc`, generic OpenBLAS) |
| **Vector** | RVV in OpenBLAS / FFTW / apps — and the bugs we fixed |
| **Custom** | X60 **IME** (`smt.vmadot`) for int8/int4 GEMM |
| **GPU** | PowerVR present; **vendor GPGPU closed** on K1 (BXM-only DDK) |

## Kernels

Libs with their own ISA kernels (RVV / IME):

- [BLAS](scientific-libs/blas.html) · [BLIS](scientific-libs/blis.html) · [FFTW](scientific-libs/fftw.html) · [MLAS](scientific-libs/mlas.html)

## Derivatives

Stack probes that call into the kernels above (FlexiBLAS / FFT / portable parallel):

- [NumPy](scientific-libs/numpy.html) · [Armadillo](scientific-libs/armadillo.html) · [R](scientific-libs/r.html) · [scikit-learn](scientific-libs/sklearn.html)
- [LAPACK](scientific-libs/lapack.html) · [ELPA](scientific-libs/elpa.html) · [ScaLAPACK](scientific-libs/scalapack.html) · [PETSc](scientific-libs/petsc.html)
- [Kokkos](scientific-libs/kokkos.html) · [PLUMED](scientific-libs/plumed.html) · [ScaFaCoS](scientific-libs/scafacos.html) · [Voro++](scientific-libs/voro.html) · [OSU](scientific-libs/osu.html)

## Apps

- [HPL](apps/hpl.html) · [Quantum ESPRESSO](apps/qe.html) · [GROMACS](apps/gromacs.html) · [LAMMPS](apps/lammps.html)
- [ONNX Runtime](apps/onnx.html) · [llama.cpp](apps/llamacpp.html)
- [OpenFOAM](apps/openfoam.html) · [waLBerla](apps/walberla.html) · [ESPResSo](apps/espresso.html)
- [MetalWalls](apps/metalwalls.html) · [MODFLOW](apps/modflow.html) · [GCC](scientific-libs/gcc.html)

## Boards

- **[VisionFive 2](boards/VisionFive2.html)** — 4× U74; HPL **3.13 → 5.28 GFLOP/s**
- **[Orange Pi RV2](boards/RV2.html)** — 8× X60 (RVV + IME); HPL **nan → 10.53**; IME / LLM notes
- **[Banana Pi F3](boards/F3.html)** — same K1 SoC, 3.7 GB RAM; HPL **11.52**; IME **~45 GOP/s**

Use the menu for full pages. Benchmarks live in [opensolvers/benchmarks](https://github.com/opensolvers/benchmarks).

## Contact

Use the **message box below** for questions or collaboration — we reply by email.

- **Public / bugs:** [opensolvers/benchmarks issues](https://github.com/opensolvers/benchmarks/issues)
- **Sponsors:** [GitHub Sponsors](https://github.com/sponsors/opensolvers) — see [Sponsors](sponsors.html)

<div class="mm-forms">
{% include mailmoose-contact.html %}
{% include mailmoose-subscribe.html %}
</div>
