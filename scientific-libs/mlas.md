# MLAS

[MLAS](https://github.com/microsoft/onnxruntime/tree/main/onnxruntime/core/mlas) (Microsoft Linear Algebra Subprograms) is ONNX Runtime's in-tree linear-algebra layer. On RISC-V SpaceMiT X60 it exposes an **IME** (`smt.vmadot`) backend for int4 **QNBit** GEMM — the kernel behind ONNX Runtime's `MatMulNBits` nodes.

Benchmark source: [opensolvers/benchmarks/onnx](https://github.com/opensolvers/benchmarks/tree/main/onnx) — [`bench_qnbit_mlas.cpp`](https://github.com/opensolvers/benchmarks/blob/main/onnx/bench_qnbit_mlas.cpp) links directly against `libonnxruntime_mlas.a` and calls the same entry points as ORT's `matmul_nbits.cc`.

See also the [ONNX Runtime app](../apps/onnx.html) for end-to-end int4 LLM-FFN inference and the [IME microbenchmarks](../boards/RV2.html#ime-integer-matrix-extension) on the Orange Pi RV2. Related (llama.cpp path): [IME1 scale-build +4.3%](../boards/RV2.html#ime1-scale-build-prefill-optimization-llamacpp).

## What it probes

[`../ime`](https://github.com/opensolvers/benchmarks/tree/main/ime) benchmarks raw `smt.vmadot` int8 GEMM. MLAS sits one layer up: **4-bit symmetric weights**, `BlkLen=32`, external per-block scales — the packing and compute path ORT uses for quantised LLM weights.

The harness calls:

- `MlasQNBitGemmPackQuantBData` — pack quantised B (nbits=4, no zero-point),
- `MlasQNBitGemmBatch<float>` — run the GEMM,

with `thread_pool=nullptr` so results are **single-thread kernel rates**, not ORT graph throughput.

## Single-thread kernel rates (X60, one core)

| Shape (M×K×N) | Kernel rate | packed B |
| ------------- | ----------: | -------: |
| 1 × 4096 × 11008 | **0.47 GOP/s** | 22.5 MB |
| 1 × 11008 × 4096 | **0.39 GOP/s** | 22.5 MB |
| 1 × 4096 × 4096 | **0.48 GOP/s** | 8.4 MB |

These isolate kernel efficiency. Production [ONNX](../apps/onnx.html) inference fans the same kernel across 8 cores (~**6×** scaling at M=1 decode).

## RISC-V pack gotcha

The X60 IME backend registers only the plain `SQ4BitGemmPackQuantBData` (`memcpy`), not the x64-style `...AndBlkSum` variant. Scales stay external and are passed at compute time via `QuantBScale`. A second "finalize" pack call with `QuantBData=nullptr` (x64 recipe) `memcpy`s from `NULL` and segfaults — the harness does a single data-only pack.

## Kernel tuning note

Once the correct ORT compute path was confirmed (`accuracy_level=4` → `SQNBIT_CompInt8`), a hand-written RVV gemv "fast path" inside the IME kernel was **28% slower** than the stock `smt.vmadot` tile path at M=1 and was reverted. Config beat micro-optimisation — see [ONNX app](../apps/onnx.html).

## Reproduce

```sh
make board CXX=$GCC14/bin/g++
LD_LIBRARY_PATH=$GCC14/lib64:$LD_LIBRARY_PATH ./qnbit-mlas-bench 1 4096 11008 50
```

**Toolchain:** ONNX Runtime 1.29.0, `foss/2025b`, X60 `smt.vmadot` (XsmtVdot v1.0), `-march=rv64gcv_zvl256b_zfh_zvfh`.
