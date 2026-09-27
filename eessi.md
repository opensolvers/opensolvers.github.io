---
title: EESSI on RISC-V
description: How EESSI works — CVMFS, the RISC-V software stack (dev.eessi.io/riscv), and how OpenSolvers uses it on U74, X60, and X100 boards.
permalink: /eessi.html
---

[EESSI](https://www.eessi.io/) — the **European Environment for Scientific Software Installations** — is a shared stack of scientific software (compilers, MPI, BLAS, apps) delivered the same way on laptops, clusters, and clouds. OpenSolvers uses it as the baseline for every board A/B we publish.

Official docs: [eessi.io/docs](https://www.eessi.io/docs/) · RISC-V repo: [`dev.eessi.io/riscv`](https://www.eessi.io/docs/repositories/dev.eessi.io-riscv/).

## What it is

EESSI packages a **compatibility layer** (a portable userland so binaries work across host distros) and a **software layer** (EasyBuild-built modules: GCC, OpenMPI, FlexiBLAS/OpenBLAS, FFTW, HPL, Quantum ESPRESSO, …). You mount the stack over **CernVM-FS (CVMFS)** — a read-only, content-addressed filesystem — then `module load` what you need. Same modules on a desk SBC and on a cluster node.

Why that matters for RISC-V work:

- **Reproducible baselines** — stock EESSI OpenBLAS / `foss` vs a patched EasyBuild install, swapped with FlexiBLAS (no app rebuild).
- **A real toolchain** — EESSI **GCC 14.3** (fixed-length RVV, modern `-march`) instead of whatever the board OS ships (often GCC 13).
- **Upstream path** — fixes we prove on boards (e.g. OpenBLAS `gemv_n`) go back as EasyBuild / EESSI updates.

## How it works on RISC-V

RISC-V support is still a **development** stack. Two CVMFS repos cooperate:

| Repo | Role on RISC-V |
| ---- | -------------- |
| `/cvmfs/software.eessi.io` | Production: **compat layer** + **init / Lmod**. Not the app binaries. |
| `/cvmfs/dev.eessi.io/riscv` | Dev software layer: GCC, EasyBuild, `foss`, OpenBLAS, HPL, … under `…/linux/riscv64/generic/` |

You always **init from production**; it hands software off to the RISC-V dev repo. Seeing “this production version only provides a RISC-V compatibility layer…” is **expected**, not a misconfiguration.

```bash
export EESSI_VERSION_OVERRIDE=2025.06-001
source /cvmfs/software.eessi.io/versions/2025.06/init/lmod/bash
# → Module for EESSI/2025.06 loaded
# → selected CPU target: riscv64/generic
```

Proof the tools come from the RISC-V repo:

```bash
command -v eb
# …/dev.eessi.io/riscv/versions/2025.06-001/software/linux/riscv64/generic/software/EasyBuild/…/eb
```

Today every RISC-V host lands on **`riscv64/generic`** — one portable build for all cores (U74, X60, X100). CPU-optimised trees (`spacemit/x60`, profile paths like `rva23u64`, …) are still in progress upstream; until then we force OpenBLAS `TARGET=` / local `-march` when measuring core-specific kernels.

**CVMFS on RISC-V:** there is often no distro package — build the client from source (or use a known-good `.deb` for your glibc/libfuse), install `cvmfs-config-eessi`, and set `CVMFS_HTTP_PROXY=DIRECT` (or a local Squid) in `/etc/cvmfs/default.local`. Status: [status.eessi.io](https://status.eessi.io).

## Extending the stack

[`EESSI-extend`](https://www.eessi.io/docs/) layers a writable install tree on top of CVMFS. That is how we ship patched OpenBLAS, U74-tuned kernels, and experiment modules without touching the read-only mirror:

```bash
module load EasyBuild/5.3.0
export EESSI_USER_INSTALL="$HOME/eessi-extend"   # must exist
module load EESSI-extend/2025.06-easybuild
eb --from-pr <num> --robot                 # EasyBuild PR → local module
flexiblas add myblas … && flexiblas default myblas
```

Same `xhpl` / `pw.x` binary; only the BLAS (or FFT) backend changes — the methodology behind [HPL](apps/hpl.html), [BLAS](scientific-libs/blas.html), and the [EESSI X60 blog](https://www.eessi.io/docs/blog/2026/07/12/risc-v-x60-openblas-hpl/).

## On our boards

| Board | Cores | What EESSI gives us |
| ----- | ----- | ------------------- |
| [VisionFive 2](boards/VisionFive2.html) | 4× U74 | Scalar baseline; U74-tuned OpenBLAS → HPL **1.69×** |
| [Orange Pi RV2](boards/RV2.html) | 8× X60 | Stock RVV + FlexiBLAS A/B; `gemv_n` fix; IME via local asm |
| [Banana Pi F3](boards/F3.html) | 8× X60 | Same K1 stack, tighter RAM |
| [Banana Pi SM10](boards/SM10.html) | 8× X100 + 8× A100 | Same init; `archdetect` → **x100**; RVA23 / IME2 work next |
| [BeagleV-Ahead](boards/Ahead.html) | 4× C910 | Not loaded yet. Factory Yocto; C910 vector is draft **0.7.1**, so the RVV 1.0 `riscv64/generic` tree is the wrong ISA |

Highlights that started as “stock EESSI vs fixed module”:

- **[HPL](apps/hpl.html)** — NaN residual on X60 → patched OpenBLAS; U74 DGEMM lift on VisionFive 2
- **[BLAS](scientific-libs/blas.html)** / **[ELPA](scientific-libs/elpa.html)** / **[QE](apps/qe.html)** — same FlexiBLAS swap story
- **[GCC](scientific-libs/gcc.html)** — EESSI GCCcore is what we patch for `-mtune=spacemit-x60`

## Further reading

- [EESSI docs](https://www.eessi.io/docs/) · [`dev.eessi.io/riscv`](https://www.eessi.io/docs/repositories/dev.eessi.io-riscv/)
- Blog: [Chasing a NaN — RVV HPL on SpaceMiT X60](https://www.eessi.io/docs/blog/2026/07/12/risc-v-x60-openblas-hpl/)
- Video: [NaN Linpack on RISC-V (EESSI)](https://www.youtube.com/watch?v=W_-8cKA-CCU) · [all videos](videos.html)
- Benchmarks: [opensolvers/benchmarks](https://github.com/opensolvers/benchmarks)
