---
title: OSU Micro-Benchmarks on SpaceMiT X60
description: On-node OpenMPI / UCX shared-memory baseline on Orange Pi RV2 — ~1.12 μs latency, ~2 GB/s uni BW (np=2); collectives at np=8.
---

# OSU Micro-Benchmarks

[OSU Micro-Benchmarks](https://mvapich.cse.ohio-state.edu/benchmarks/) calibrate MPI interconnect performance. On the Orange Pi RV2 we measure **on-node** OpenMPI shared-memory (self+vader) so MPI-heavy apps ([HPL](../apps/hpl.html), [ScaLAPACK](scalapack.html), [QE](../apps/qe.html), [PETSc](petsc.html)) can be read against a known latency/bandwidth floor.

Benchmark source: [opensolvers/benchmarks/osu](https://github.com/opensolvers/benchmarks/tree/main/osu). Module: `OSU-Micro-Benchmarks/7.5.1-gompi-2025b`.

> **Bottom line:** sub-2 μs shared-memory latency and ~**2 GB/s** uni-directional BW — pure MPI overhead is small next to BLAS/FFT work on this board. Cross-node Ethernet needs a second machine (not measured here).

---

## Results (2026-08-21)

OpenMPI 5.0.8, `pml=ob1 btl=self,vader`, `--bind-to core`.

### Point-to-point (np=2)

| Metric | Value |
| ------ | ----: |
| Latency @ 1 B | **1.12 μs** |
| Latency @ 1 KiB | 2.73 μs |
| Latency @ 1 MiB | 640.56 μs |
| Uni-directional BW peak | **2069 MB/s** (@ 512 KiB) |
| Bi-directional BW peak | **2456 MB/s** (@ 256 KiB) |

### Collectives (np=8, full X60)

| Test | latency @ 4 B / 1 KiB / 1 MiB |
| ---- | ---------------------------- |
| `osu_allreduce` | 6.96 / 18.51 / 13410 μs |
| `osu_bcast` | 3.04 / 8.86 / 3758 μs |
| `osu_alltoall` | 10.97 / 41.90 / 24411 μs |

When an 8-rank ScaLAPACK/PETSc run looks communication-bound, compare message sizes to the collective table — `alltoall` at ≥64 KiB is already multi-millisecond per call.

**Measured:** 2026-08-21 on Orange Pi RV2.
