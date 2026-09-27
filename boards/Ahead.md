---
title: BeagleV-Ahead — T-Head C910
description: BeagleV-Ahead (TH1520) — quad Xuantie C910 at 1.85 GHz, 3-wide out-of-order, draft RVV 0.7.1 (xtheadvector, 128-bit), GhostWrite, plus the on-chip C906, 4 TOPS NPU, and BXM-4-64.
---

The BeagleV-Ahead is a BeagleBone-sized board on the T-Head **TH1520**. The CPUs that run Linux are four **Xuantie C910** cores. Beagle’s board manual clocks that cluster at **1.85 GHz**; the [product page](https://www.beagleboard.org/boards/beaglev-ahead) lists **2 GHz**. Ours is on the desk on the factory image: **THEAD C910 Release Distro 1.1.2**, kernel `5.10.113-yocto-standard` (built 2023-06-10). **4 GB** LPDDR4, **16 GB** eMMC.

Product page: [BeagleV-Ahead](https://www.beagleboard.org/boards/beaglev-ahead) · [docs](https://docs.beagleboard.org/boards/beaglev/ahead/index.html).

## The CPU: Xuantie C910

C910 is T-Head’s first out-of-order application core, and the open RTL behind the TH1520 cluster (up to four cores sharing an L2). On this SoC that cluster is the whole Linux SMP set.

| | **C910** on the Ahead |
| --- | --- |
| Count | **4** (one cluster) |
| Clock | **1.85 GHz** in the board manual; product page says **2 GHz** |
| Pipeline | **3-wide**, **12-stage**, out-of-order issue and completion, in-order retirement; **8** execution ports |
| L1 | **64 KB** I + **64 KB** D per core |
| L2 | **1 MB** shared, inclusive |
| Scalar ISA | **RV64GC** |
| Vector | Draft **RVV 0.7.1**, exposed as **`xtheadvector`**, **128-bit** datapath |

Fetch is in order. Issue, execution, and completion are out of order, then retirement is in order again. The integer side has a dedicated branch port plus ALU ports; the FP/vector side is a dual pipe, and both pipes can do **128-bit** vector ops. The vector register file supplies the mask as well — RVV has no separate mask registers.

That vector unit is the part that does not match the rest of this site. X60 and X100 implement **ratified RVV 1.0** (`Zvl256b` on those chips, VLEN=256). C910 implements the **0.7.1 draft**. The encodings and the programming model differ (`xtheadvector` versus `v`). A binary built with `-march=rv64gcv` for the Orange Pi RV2 or the BPI-SM10 does not run on this core. Tooling has to target T-Head’s vector extension explicitly.

**GhostWrite** ([CVE-2024-44067](https://nvd.nist.gov/vuln/detail/CVE-2024-44067)) is a bug in that same vector unit: a faulty store uses a physical address, so an unprivileged process can write memory outside its own mapping. It is a hardware erratum on the C910 in the TH1520 (and on the related C920). The mitigation in later kernels is to **turn `xtheadvector` off**. The factory 5.10 image on this board predates that erratum. With the vector unit disabled, the C910 that remains is the 3-wide scalar RV64GC core — the 128-bit pipes stay dark.

## Other compute on the same chip

These are not the Linux CPUs. They sit beside the C910 cluster and have their own drivers.

| Block | What it is |
| --- | --- |
| **C906** | One smaller T-Head core for the **audio** subsystem. It does not join the 4-core SMP set. |
| **NPU** | **4 TOPS INT8** at 1 GHz. Vendor neural path (TensorFlow / ONNX / Caffe in T-Head’s stack), separate from the C910 vector pipes. |
| **GPU** | Imagination **BXM-4-64**. Beagle rates it around **50 GFLOPS**, with GLES / Vulkan / OpenCL via the usual PowerVR stack. Same GPU family as the [BPI-SM10](SM10.html), different SoC and driver story. |

## Where this sits next to the other boards

| Board | Application cores | Vector |
| --- | --- | --- |
| [VisionFive 2](VisionFive2.html) | 4× SiFive U74, in-order | none |
| **BeagleV-Ahead** | 4× C910, out-of-order | draft RVV 0.7.1, 128-bit |
| [Orange Pi RV2](RV2.html) / [BPI-F3](F3.html) | 8× X60 | RVV 1.0, VLEN=256, plus IME1 |
| [BPI-SM10](SM10.html) | 8× X100 + 8× A100 | RVV 1.0, VLEN=256 / 1024, plus IME2 |

## Status

Console login works on the factory Yocto image. [EESSI](../eessi.html) is not on this board yet: `dev.eessi.io/riscv` ships **RVV 1.0** userspace, which is the wrong vector ISA for a C910. Benchmark pages will land here once there is a userspace that can build `xtheadvector` and the same A/B method as [RV2](RV2.html).

## Related

- [BeagleV-Ahead](https://www.beagleboard.org/boards/beaglev-ahead) — product page
- [Beagle docs](https://docs.beagleboard.org/boards/beaglev/ahead/index.html)
- [GhostWrite](https://ghostwriteattack.com/) — CVE-2024-44067
- [Chips and Cheese: Xuantie C910](https://chipsandcheese.com/p/alibabat-heads-xuantie-c910) — pipeline and caches on the TH1520
- [opensolvers/benchmarks](https://github.com/opensolvers/benchmarks)
