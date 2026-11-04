---
title: llama.cpp on RISC-V — X60 IME vs RVV
description: llama.cpp on RISC-V. K3 IME2 prompt 526 t/s on Qwen2.5-0.5B Q4_0. RV2 IME wins prompt from 1.1B up; RVV wins decode.
---

# llama.cpp

[llama.cpp](https://github.com/ggml-org/llama.cpp) runs local GGUF models. We measure it on the [Orange Pi RV2](../boards/RV2.html) (X60, IME1) and the [Banana Pi SM10](../boards/SM10.html) (K3, IME2). Fork: [`opensolvers/llama.cpp`](https://github.com/opensolvers/llama.cpp) branch [`x60-ime-rvv`](https://github.com/opensolvers/llama.cpp/tree/x60-ime-rvv). Raw notes: [benchmarks/llamacpp](https://github.com/opensolvers/benchmarks/tree/main/llamacpp).

**Prompt** and **decode** are tokens/s. Unless a column says otherwise, that is 128 prompt tokens and 32 generated tokens (`llama-bench -p 128 -n 32`).

## K3 — Banana Pi SM10

<a id="results-on-the-k3"></a>
<a id="measured--ime2-banana-pi-sm10-q4_0"></a>
<a id="measured--ime2-banana-pi-sm10-q3_k_m"></a>
<a id="measured--ime2-banana-pi-sm10-tinyllama-q2_k"></a>

Eight A100 cores at 1.8 GHz, `use_ime2: 1`. Q4_0 is r=3 at fork `72d1824`. Q3_K_M and TinyLlama are r=3 at `107ead2f2`.

| | 1 core | 2 cores | 4 cores | 8 cores |
| --- | -----: | -----: | -----: | -----: |
| Q4_0 prompt | 76.43 | 149.65 | 291.12 | **525.96** |
| Q4_0 decode | 21.90 | 40.24 | **58.67** | 38.78 |
| Q3_K_M prompt | 69.04 | — | 266.35 | **499.23** |
| Q3_K_M decode | 19.14 | — | 45.56 | **47.95** |
| TinyLlama prompt | 35.73 | — | 138.25 | **254.37** |
| TinyLlama decode | 10.93 | — | 35.13 | **43.39** |

Q4_0 decode peaks at 4 cores. Q3_K_M and TinyLlama peak at 8. The same Q4_0 file on the X100 (RVV, 8 cores) is **193.68 / 41.95**, so IME2 prompt is **2.7×** that core. On the RV2 the same file at 8 cores is IME 69.03 / 1.47 and RVV 100.60 / 12.18.

The Q3_K_M file is mostly IQ4_NL, not Q3_K. Its 8-core prompt is 5% under Q4_0. TinyLlama is tagged Q2_K; most of its matmuls are Q3_K, and this prompt rate is **1.24×** the previous kernel (205.17). Stock RVV on the A100 has no VLEN=1024 Q4_0 kernel and lands at 6.68 / 4.27. Notes: [Q4_0](https://github.com/opensolvers/benchmarks/blob/main/llamacpp/results_sm10_q4_0.md), [Q3_K_M](https://github.com/opensolvers/benchmarks/blob/main/llamacpp/results_sm10_q3_k_m.md), [TinyLlama](https://github.com/opensolvers/benchmarks/blob/main/llamacpp/results_sm10_tinyllama_q2_k.md).

## Ten Q4_0 models

<a id="measured--ime2-banana-pi-sm10-q4_0-suite"></a>
<a id="measured--ime-vs-rvv-orange-pi-rv2"></a>

Eight cores on both boards, same bench. RV2 is the X60 (IME1 and RVV). SM10 is the K3 A100 (IME2), one repeat at fork `107ead2f2`. **×** is the SM10 result divided by the faster RV2 result on that row. Prompt is about **5–8×**. Decode is about **3–5×**.

On the RV2, IME wins prompt from 1.1B up and RVV wins every decode, so the prompt × uses IME except on the 0.5B, and every decode × uses RVV. The RV2 has 7.7 GiB and no swap, so 7B is the practical ceiling there. **Qwen2.5 3B on the SM10 repeated G**, so that row is a failed run. Qwen2.5 7B is the two-shard file. The same 3B model at Q4_K_M is coherent.

The 0.5B SM10 cells are one repeat. The r=3 mean for that file is prompt 525.96 and decode 38.78 at 8 cores, with decode peaking at 58.67 on 4 cores.

| Model | Prompt RV2 IME | Prompt RV2 RVV | Prompt SM10 | × | Decode RV2 IME | Decode RV2 RVV | Decode SM10 | × |
| ----- | -------------: | -------------: | ----------: | -: | -------------: | -------------: | ----------: | -: |
| Qwen2.5 0.5B | 69.03 | 100.60 | 540.21 | 5.4× | 1.47 | 12.18 | 57.78 | 4.7× |
| TinyLlama 1.1B | 37.72 | 34.95 | 279.67 | 7.4× | 1.72 | 9.47 | 35.26 | 3.7× |
| Llama 3.2 1B | 38.87 | 25.27 | 291.49 | 7.5× | 2.00 | 7.74 | 25.86 | 3.3× |
| Qwen2.5 1.5B | 27.14 | 22.77 | 204.11 | 7.5× | 1.19 | 6.10 | 22.83 | 3.7× |
| Gemma 2 2B | 17.56 | 10.04 | 114.81 | 6.5× | 1.01 | 3.15 | 10.53 | 3.3× |
| Qwen2.5 3B (failed) | 14.13 | 7.85 | 103.75 | 7.3× | 0.85 | 3.12 | 12.32 | 3.9× |
| Llama 3.2 3B | 13.99 | 6.95 | 108.70 | 7.8× | 1.05 | 3.09 | 11.72 | 3.8× |
| Phi-3.5 mini | 11.37 | 4.59 | 78.99 | 6.9× | 1.11 | 2.02 | 10.48 | 5.2× |
| Mistral 7B | 5.84 | 2.35 | 45.57 | 7.8× | 0.73 | 1.38 | 5.75 | 4.2× |
| Qwen2.5 7B | 6.06 | 2.52 | 48.81 | 8.1× | 0.73 | 1.38 | 5.70 | 4.1× |

Sources: [model_validation.tsv](https://github.com/opensolvers/benchmarks/blob/main/llamacpp/model_validation.tsv), [results_sm10_model_suite.md](https://github.com/opensolvers/benchmarks/blob/main/llamacpp/results_sm10_model_suite.md).

## Seventeen Q4_K_M models

<a id="measured--ime2-banana-pi-sm10-q4_k_m-suite"></a>
<a id="study-2--q4_k_m-17-models-m1gemv-vs-rvv"></a>

Same files. **×** is the SM10 result divided by the faster RV2 result on that row, as in the table above. RV2 columns are **512** prompt tokens and **64** generated tokens (IME1 `m1gemv` vs RVV). SM10 columns are **128** and **32** (IME2, one repeat). Prompt × is about **7–18×** and decode × is about **3–6×**. The prompt factor is larger because the RV2 prompt is the longer one. On the RV2, RVV wins every decode and prompt is close; Q4_K does not engage `smt.vmadot` there. On the SM10, Q4_K uses the IME2 Q4 kernel and all 17 completions are coherent. StableLM 2 1.6B is the Zephyr GGUF. DeepSeek Coder described RISC-V as an ARM ISA, and the DeepSeek-R1 distill opened with a thinking trace.

| Model | Prompt RV2 IME | Prompt RV2 RVV | Prompt SM10 | × | Decode RV2 IME | Decode RV2 RVV | Decode SM10 | × |
| ----- | -------------: | -------------: | ----------: | -: | -------------: | -------------: | ----------: | -: |
| Qwen2.5 0.5B | 23.92 | 20.73 | 433.34 | 18.1× | 1.42 | 9.19 | 51.44 | 5.6× |
| Llama 3.2 1B | 24.13 | 27.49 | 219.68 | 8.0× | 1.90 | 8.19 | 24.49 | 3.0× |
| Gemma 3 1B | 12.75 | 11.36 | 191.69 | 15.0× | 1.04 | 4.40 | 22.63 | 5.1× |
| DeepSeek Coder 1.3B | 16.51 | 16.43 | 172.65 | 10.5× | 1.37 | 6.91 | 25.31 | 3.7× |
| Qwen2.5 1.5B | 18.62 | 18.80 | 147.25 | 7.8× | 1.14 | 6.31 | 20.09 | 3.2× |
| StableLM 2 Zephyr 1.6B | 16.58 | 20.65 | 167.67 | 8.1× | 1.17 | 6.74 | 21.19 | 3.1× |
| Falcon 3 1B | 19.57 | 22.49 | 179.10 | 8.0× | 1.69 | 7.28 | 21.78 | 3.0× |
| SmolLM2 1.7B | 13.62 | 15.46 | 132.45 | 8.6× | 1.35 | 5.94 | 18.90 | 3.2× |
| DeepSeek R1 Qwen 1.5B | 18.54 | 19.10 | 147.74 | 7.7× | 1.14 | 6.38 | 20.06 | 3.1× |
| Granite 3.1 2B | 9.23 | 10.48 | 86.95 | 8.3× | 0.80 | 4.04 | 12.98 | 3.2× |
| Gemma 2 2B | 11.43 | 12.07 | 87.09 | 7.2× | 0.97 | 3.25 | 9.07 | 2.8× |
| Qwen2.5 3B | 8.25 | 8.93 | 73.31 | 8.2× | 0.79 | 3.43 | 10.69 | 3.1× |
| Falcon 3 3B | 10.37 | 10.68 | 82.71 | 7.7× | 1.17 | 3.74 | 9.94 | 2.7× |
| Llama 3.2 3B | 8.59 | 9.14 | 77.84 | 8.5× | 0.96 | 3.30 | 10.37 | 3.1× |
| Phi-3.5 mini | 4.08 | 2.91 | 50.79 | 12.4× | 0.69 | 2.04 | 9.25 | 4.5× |
| Gemma 3 4B | 7.67 | 7.68 | 58.17 | 7.6× | 0.70 | 2.52 | 7.48 | 3.0× |
| Phi-4 mini | 5.33 | 4.14 | 60.76 | 11.4× | 0.76 | 2.17 | 8.39 | 3.9× |

Sources: [results_17model_m1gemv_vs_rvv.md](https://github.com/opensolvers/benchmarks/blob/main/llamacpp/results_17model_m1gemv_vs_rvv.md), [results_sm10_model_suite.md](https://github.com/opensolvers/benchmarks/blob/main/llamacpp/results_sm10_model_suite.md). The BXM-4-64 GPU is still unmeasured.

## Build

On an X60, with an assembler that accepts `smt.vmadot` ([toolchain](../boards/RV2.html#toolchain-support-xsmtvdot)):

```bash
git clone -b x60-ime-rvv https://github.com/opensolvers/llama.cpp.git
cd llama.cpp
cmake -B build -DCMAKE_BUILD_TYPE=Release \
  -DGGML_CPU_RISCV64_SPACEMIT=ON \
  -DGGML_RVV=ON -DGGML_RV_ZVFH=ON -DGGML_RV_ZFH=ON \
  -DGGML_RV_ZBA=ON -DGGML_RV_ZICBOP=ON
cmake --build build --parallel "$(nproc)"
```

Pin IME to cluster 0 (`taskset -c 0-3`). Startup should report `use_ime1: 1`. A second build with `GGML_CPU_RISCV64_SPACEMIT=OFF` is the RVV arm. Branch docs: [build-riscv64-spacemit.md](https://github.com/opensolvers/llama.cpp/blob/x60-ime-rvv/docs/build-riscv64-spacemit.md). Harnesses: [benchmarks/llamacpp](https://github.com/opensolvers/benchmarks/tree/main/llamacpp).

On the K3, write the shell pid to `/proc/set_ai_thread` before `exec`, then `taskset -c 8-15`. See [SM10](../boards/SM10.html#getting-onto-a100-procset_ai_thread).

## Kernels on the fork

| Change | Result |
| ------ | ------ |
| [Q8_0 / Q6_K GEMM](https://github.com/opensolvers/llama.cpp/pull/1) | Q8_0 prompt 62.7 t/s at 4 cores, vs 25.5 on the stock IME build |
| [Scale-build](https://github.com/opensolvers/llama.cpp/pull/2) | +4.3% on an isolated Q4_0 GEMM (22.50 → 23.47 GOP/s) |
| [`xsmtvdot` asm](https://github.com/opensolvers/llama.cpp/pull/3) | Builds with binutils ≥ 2.46 |
| [RVV softmax](https://github.com/opensolvers/llama.cpp/pull/4) | 212/212 tests; about 2.1× on a large shape |
| [M1 GEMV](https://github.com/opensolvers/llama.cpp/pull/5) | Q8_0 decode 0.82 → 5.29 t/s. On RV2 Q4_K_M this build is slower than RVV |

**Q8_0 hybrid** stores weights twice and uses IME only when the row count is at least 4. Decode goes back to the stock path. `SPACEMIT_HYBRID=1`.

| Build | Q8_0 prompt | Q8_0 decode |
| ----- | ----------: | ----------: |
| IME only | 83.7 | 0.83 |
| Hybrid | 90.1 | 6.68 |
| RVV only | 27.0 | 5.11 |

**TCM off on the RV2.** Staging panels through TCM slows the real model: Q8_0 prompt 70.1 → 50.4, Q4_0 prompt 96.8 → 49.9. Keep `SPACEMIT_DISABLE_TCM=1`. Details: [RV2](../boards/RV2.html#how-to-use-ime-and-when-not-to-use-tcm).
