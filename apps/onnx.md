---
title: ONNX Runtime on X60 — real LLM decode via IME MatMulNBits
description: Orange Pi RV2 story — Qwen, SmolLM2, and TinyLlama int4/int8 decode through ORT MLAS CompInt8; from accuracy_level=4 to Q4×16 and SQ8 Q8×16 panels.
---

# ONNX Runtime

**Video (part 1):** [10× ONNX on RISC-V: One Missing Attribute Unlocks X60 IME](https://www.youtube.com/watch?v=IV3TV57eGAs) — [all videos](../videos.html)

[ONNX Runtime](https://onnxruntime.ai/) runs ONNX graphs end-to-end. On the SpaceMiT X60 we care about quantized LLM layers: Microsoft **`MatMulNBits`** through MLAS `SQNBit`, driven by the X60 **IME** (`smt.vmadot`).

Benchmark source: [opensolvers/benchmarks/onnx](https://github.com/opensolvers/benchmarks/tree/main/onnx) — m1pack IME patches, `run_real_llm_ort.cpp`, and model harnesses. Kernel microbench: [MLAS](../scientific-libs/mlas.html). Raw IME: [RV2](../boards/RV2.html#ime-integer-matrix-extension). Same silicon in GGML: [llama.cpp](llamacpp.html).

---

## Results first — real decode on the Orange Pi RV2

All numbers below are **greedy decode** through a patched ORT 1.29.0 (`accuracy_level=4` → CompInt8 IME), not synthetic FFN-only graphs. Decode is ms/token; lower is better. Pin to cluster 0 for IME (`taskset -c 0` / `0-3`).

### Across models (decode)

| Model | Quant | BlkLen | 1 thread | 4 threads |
| ----- | ----- | -----: | -------: | --------: |
| **Qwen2.5-0.5B** | int4 | 32 | ~108 ms | **~80 ms** |
| **Qwen2.5-0.5B** | int8 | 32 | ~241 ms | **~159 ms** |
| **SmolLM2-360M** | int4 | 32 | — | **~80 ms** |
| **SmolLM2-360M** | int8 | 32 | ~198 ms | **~140 ms** |
| **TinyLlama-1.1B** | int4 | 32 | — | **~156 ms** |

Int8 is ~1.7–2× slower than int4 on the same model — expected (roughly 2× weight traffic) — but both paths now hit IME instead of a multi-second generic fallback.

### Qwen2.5-0.5B — how far we came

Same board, same prompt style, real GenAI ONNX graphs:

| Path | Decode (1t) | Decode (4t) | Notes |
| ---- | ----------: | ----------: | ----- |
| CompFp32 fallback (acc ≠ 4) | **~16 s/tok** | — | AMD BlkLen=128 export, wrong MLAS path |
| IME CompInt8 BlkLen=128 | **~0.93 s/tok** | — | **~17×** vs CompFp32; first real Qwen win |
| IME Q4×16 panels (BlkLen=32) | ~108 ms | **~80 ms** | pack-time Q4_0×16 + llama M1 asm |
| Int8 before SQ8 IME | ~12.8 s | ~4.4 s | bits=8 had no CompInt8 kernel |
| Int8 SQ8 IME (flat pack) | ~1216 ms | ~378 ms | first `smt.vmadot` int8 path |
| **Int8 Q8×16 panels** | **~241 ms** | **~159 ms** | Llama i8i8 M1 panels |

Prefill on the BlkLen=128 Qwen path dropped from ~18 s → ~3.9 s (~4.7×) once CompInt8 was selected.

### Synthetic FFN (context)

The early probe was an 8-layer Llama-7B-shaped FFN stack (16 `MatMulNBits`, BlkLen=32). Stock CompInt8 after `accuracy_level=4`: **~3.5 s** (x1). With m1pack Q4×16: **~139 ms**. That micro story unlocked the model work above; it is not the headline anymore.

---

## How we got there

### 1. Prove the kernel runs — `accuracy_level=4`

ORT picks the compute variant in `matmul_nbits.cc`:

| `accuracy_level` | MLAS path | On X60 |
| ---------------- | --------- | ------ |
| **4** | `SQNBIT_CompInt8` | IME `smt.vmadot` |
| anything else | `SQNBIT_CompFp32` | *no IME kernel* → fp32 dequant + SGEMM |

Our first generated FFN had **zero** `accuracy_level` attributes. Roofline said we were 178–341× above the STREAM bandwidth floor — wrong path, not a slow kernel. A one-shot probe in the IME kernel never fired. Setting `accuracy_level=4` (or [`patch_accuracy_level.py`](https://github.com/opensolvers/benchmarks/blob/main/onnx/patch_accuracy_level.py), including rewriting `0→4` on AMD exports) selected CompInt8 with **no kernel change** — **9.1× / 10.3×** on the synthetic FFN (x1 / x8).

A hand-written M&lt;4 RVV “fast path” inside the IME kernel then measured **28% slower** than the stock `smt.vmadot` tile and was reverted. Config beat micro-opts.

### 2. Pack for decode — Q4×16 m1pack (BlkLen=32)

Stock CompInt8 still left decode bandwidth on the table. The ship path packs weights once into **Q4_0×16 panels** (llama-cpp layout) and runs dedicated **M=1** asm plus M≥4 gather — see [`vendor/sqnbitgemm_kernel_ime.m1pack.cpp`](https://github.com/opensolvers/benchmarks/blob/main/onnx/vendor/sqnbitgemm_kernel_ime.m1pack.cpp). Isolated microbench reaches **~10 GOP/s** single-thread; Qwen / SmolLM2 BlkLen=32 decode lands in the **~80 ms/tok @4t** row of the results table.

### 3. Match real exports — BlkLen=128 CompInt8

AMD’s Qwen2.5-0.5B package uses **`block_size=128`**, not 32. The Q4×16 panel path does not apply. We added a BlkLen=128 CompInt8 path (column-major pack + RVV nibble gather + IME) so `accuracy_level=4` actually runs IME on that graph — the **~17×** decode win vs CompFp32. (A truncated RVV unpack that only processed 64 of 128 nibbles produced garbage logits until fixed.)

### 4. Int8 was a lose until SQ8Bit IME

`bits=8` MatMulNBits had no X60 CompInt8 backend — Qwen int8 sat at **multi-second** tokens. We added **SQ8Bit CompInt8**: pack signed `B′ = B−128`, flat scales, width-16 block sums (`−scale·(zp−128)`, zero when GenAI omits ZP / zp=128), kernel via `smt.vmadot`. That alone brought Qwen int8 to ~1.2 s / ~0.4 s (1t / 4t).

### 5. Close the gap — Q8×16 M1 panels

Mirror the int4 panel trick for int8: Llama **i8i8** layout (16 columns × k-block: fp16 scales + tiles; column stride 34), M1 asm `gemm_m1_panel_q8x16`, M≥4 gather from panels, and a resolve helper so BlkSum/scales sit **after** the panel slab (struct layout alone is wrong). Qwen int8 drops to **~241 / ~159 ms**; SmolLM2 int8 to **~140 ms @4t**. SQ4 regression on SmolLM2 / TinyLlama stayed flat (~80 / ~156 ms @4t).

GenAI int8 exports often mis-wire the embed path (`weight_Q4`, `bits=4`, half-width reshape). Fix to `weight_Q8` / `bits=8` / full hidden before load — same class of “grep the artifact” bug as missing `accuracy_level`.

### 6. Panel loop (pre-TCM)

Port of the ime-bench step-2 lever into MLAS CompInt8: **N-panel outer** (16 cols, offline Q4×16 / Q8×16) → M-inner → K, with driver `StrideN=16` for `M≥4`. M=1 llama M1 asm keeps wide chunks.

| Shape | m1pack | +panel | Δ |
| ----- | -----: | -----: | --: |
| 1×4096×11008 | 9.45 GOP/s | 9.45 | — |
| **4×4096×11008** | 2.10 | **2.51** | **+19%** |

Script: [`run-qnbit-panel-ab.sh`](https://github.com/opensolvers/benchmarks/blob/main/onnx/run-qnbit-panel-ab.sh). Helps prefill / batched decode; M=1 decode unchanged.

### 7. TCM — skip for ORT on RV2

ONNX MLAS has no `libspine_tcm` path. A/B with packed B in `/dev/tcm` ([`bench_qnbit_tcm.cpp`](https://github.com/opensolvers/benchmarks/blob/main/onnx/bench_qnbit_tcm.cpp)): offline TCM **−6…−10%** when B fits; real FFN packedB (**25 MiB**) skips. Same conclusion as llama — **IME yes, TCM no** for e2e. Guide: [RV2](../boards/RV2.html#how-to-use-ime-and-when-not-to-use-tcm).

---

## Reproduce

```sh
# On the X60 board: apply m1pack IME patches, rebuild ORT
bash apply-ime-m1pack.sh
make -C $ORT_BUILD onnxruntime_mlas onnxruntime

# Force CompInt8 on any MatMulNBits graph
python3 patch_accuracy_level.py model.onnx model_acc4.onnx

# Real decode (env sets KV shape / prompt tokens)
bash run-real-llm-ort.sh          # Qwen-shaped defaults
bash run-smollm2-ort.sh           # SmolLM2-360M
bash run-tinyllama-ort.sh         # TinyLlama-1.1B
```

**Toolchain:** ONNX Runtime 1.29.0, `foss/2025b`, X60 `smt.vmadot` (assembler-only — [toolchain notes](../boards/RV2.html#toolchain-support-xsmtvdot)), `-march=rv64gcv_zvl256b_zfh_zvfh`. Full log: [`MLAS_IME_IMPROVE.md`](https://github.com/opensolvers/benchmarks/blob/main/onnx/MLAS_IME_IMPROVE.md).

## Takeaways

1. **Results on real models first** — synthetic FFN found the path; Qwen / SmolLM2 / TinyLlama are the scoreboard.
2. **Prove the code runs** — probes + roofline beat weeks of kernel tuning on a dead path.
3. **Config, then pack, then panels** — `accuracy_level=4` → CompInt8; BlkLen must match the pack; M1 panels recover decode.
4. **Int8 needed its own ship path** — SQ8Bit + Q8×16, not “hope int4 helps.”
5. **Grep the ONNX** — missing attributes and broken embed wiring look like slow kernels until you read the bytes.
6. **Panel loop for M≥4** — N-outer B-panel +19% on 4×FFN; TCM does not help e2e on RV2.
