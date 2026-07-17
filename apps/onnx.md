# ONNX Runtime

[ONNX Runtime](https://onnxruntime.ai/) int4 **`MatMulNBits`** inference on SpaceMiT X60 — a whole-graph LLM decode probe through ORT's MLAS `SQNBit` path and the X60 **IME** (`smt.vmadot`).

Benchmark source: [opensolvers/benchmarks/onnx](https://github.com/opensolvers/benchmarks/tree/main/onnx) — Llama-7B-proportioned FFN stack, `patch_accuracy_level.py`, and `onnxruntime_perf_test` runners.

Library-level kernel numbers: [MLAS](../scientific-libs/mlas.html). Raw IME microbenchmarks: [Orange Pi RV2](../boards/RV2.html#ime-integer-matrix-extension).

## The workload

8-layer stack of FFN/projection blocks: `hidden=4096`, `ffn=11008`, **16 `MatMulNBits` nodes** (com.microsoft), 4-bit symmetric weights, `block_size=32` — the big quantised GEMMs in LLM decode. Measured with `onnxruntime_perf_test` (`-m times`, `-r 8`) at decode shape **M=1**.

## Before / after — one missing attribute

ORT selects the int4 compute variant in `matmul_nbits.cc`:

| `accuracy_level` | MLAS path | X60 backend |
| ---------------- | --------- | ----------- |
| **4** | `SQNBIT_CompInt8` | IME `smt.vmadot` kernel |
| anything else | `SQNBIT_CompFp32` | *no kernel registered* → generic fp32 dequant + SGEMM fallback |

The generated model had **zero** `accuracy_level` attributes across all 16 nodes. Setting `accuracy_level=4` at generation time (or via [`patch_accuracy_level.py`](https://github.com/opensolvers/benchmarks/blob/main/onnx/patch_accuracy_level.py)) selects the IME path — **no kernel code change**.

### End-to-end latency (Orange Pi RV2 / X60)

| Configuration | Before (CompFp32 fallback) | After (IME CompInt8) | Speedup |
| ------------- | -------------------------: | -------------------: | ------: |
| Single thread (`-x1`) | 31,956 ms | **3,522 ms** | **9.1×** |
| 8 threads (`-x8`) | 6,074 ms | **590 ms** | **10.3×** |

- Peak RSS: ~1023 MB → **842 MB** (no fp32 dequant buffers).
- x1→x8 scaling on the fixed path: **~6×** across 8 cores.

## How it was confirmed

- **Probe:** `fprintf` in both IME kernel paths — at M=1, **neither fired** on the broken model; the kernel never ran.
- **Roofline:** X60 STREAM triad = 3.85 GB/s (1T) / ~10.6 GB/s (8T); int4 B traffic ≈ 361 MB/inference → bandwidth floor ~0.09 s (x1). Measured latency was **178–341× above** the floor — wrong code path, not slow kernel.
- **Bytes:** `grep -c accuracy_level model.onnx` → 0 across 16 nodes.

After the fix, the probe fires and first inference drops from ~32 s to ~4.5 s.

## Kernel micro-opt was a regression

The original goal was an M&lt;4 RVV gemv fast path inside the IME `SQ4BitGemmKernel_CompInt8`. Once the correct path executed, fair A/B at M=1 showed:

| Kernel (M=1) | Latency |
| ------------ | ------: |
| Hand-written RVV fast path | 4,521 ms |
| Stock IME `smt.vmadot` tile | **3,522 ms** |

The bespoke path was **28% slower** and was reverted. **Config beat kernels** — one attribute gave 9–10×; the hand kernel gave −28%.

## Reproduce

```sh
python3 patch_accuracy_level.py int4_ffn.onnx int4_ffn_acc4.onnx

onnxruntime_perf_test -e cpu -I -m times -r 8 -x 1 int4_ffn_acc4.onnx
onnxruntime_perf_test -e cpu -I -m times -r 8 -x 8 int4_ffn_acc4.onnx
```

**Toolchain:** ONNX Runtime 1.29.0, `foss/2025b`, X60 `smt.vmadot` MLAS backend, `-march=rv64gcv_zvl256b_zfh_zvfh`.

## Takeaways

1. **Prove the code runs before optimising it** — a one-shot probe would have saved the kernel-tuning detour.
2. **Roofline early** — traffic math pointed at "wrong path" in minutes.
3. **Comments aren't evidence** — grep the artifact, not the intent.
4. **Config beats kernels** — `accuracy_level=4` → 9–10×; hand-tuned RVV → −28%.
