---
title: vLLM on the K3 — install and the A100 IME2 GEMM
description: vLLM on the Banana Pi SM10. Stock X100 CPU backend for ten small models, and a vmadot.hp int4 GEMM for the A100.
---

# vLLM

[vLLM](https://github.com/vllm-project/vllm) serves transformer models. The build below is the CPU backend on the [Banana Pi SM10](../boards/SM10.html). Fork: [`opensolvers/vllm`](https://github.com/opensolvers/vllm) branch [`k3-a100-ime2`](https://github.com/opensolvers/vllm/tree/k3-a100-ime2), two commits past upstream `e090987`. Other engines on this board: [ONNX Runtime](onnx.html), [llama.cpp](llamacpp.html).

## Ten models

<a id="models"></a>

X100 cores, bfloat16, eager mode, `max_model_len=192`. The prompt is 128 token ids of the sentence “The capital of France is”, decoded to text, then 32 new tokens at temperature 0. Prompt tokens/s is the prompt length over the time to the first new token. Decode tokens/s is the remaining 31 tokens over the time after that. vLLM keeps one X100 core for the scheduler and binds the worker to the other seven. TinyLlama’s tokenizer counted 129 prompt tokens after that decode-and-encode; its rate uses 129.

These runs use the stock RISC-V CPU kernels (VLEN 256).

## IME2 generation

<a id="ime2-generate"></a>

Same prompt, with the bf16 weights quantized to the int4 tile. Each matrix is packed once. Every linear layer calls `Ime2HpGemm` on eight A100 cores, one core per slice of the N tiles. Norms, RoPE and attention sit beside that call. TinyLlama again counted 129 prompt tokens.

| Model | Prompt | Decode |
| ----- | -----: | -----: |
| SmolLM2 135M | 87.11 | 11.14 |
| SmolLM2 360M | 49.80 | 8.44 |
| Qwen2.5 0.5B | 48.88 | 9.57 |
| TinyLlama 1.1B | 30.09 | 7.38 |
| Qwen2.5 1.5B | 24.98 | 6.11 |
| DeepSeek R1 distill Qwen 1.5B | 24.79 | 6.25 |
| StableLM 2 Zephyr 1.6B | 21.02 | 5.04 |
| SmolLM2 1.7B | 17.10 | 4.74 |
| Phi-2 | 14.77 | 3.66 |
| Qwen2.5 3B | 14.47 | 3.86 |

| Model | Prompt | Decode |
| ----- | -----: | -----: |
| SmolLM2 135M | 1.95 | 1.27 |
| SmolLM2 360M | 0.75 | 0.53 |
| Qwen2.5 0.5B | 0.67 | 0.43 |
| TinyLlama 1.1B | 0.23 | 0.17 |
| Qwen2.5 1.5B | 0.18 | 0.11 |
| DeepSeek R1 distill Qwen 1.5B | 0.18 | 0.11 |
| StableLM 2 Zephyr 1.6B | 0.18 | 0.12 |
| SmolLM2 1.7B | 0.12 | 0.10 |
| Phi-2 | 0.09 | 0.07 |
| Qwen2.5 3B | 0.09 | 0.06 |

## IME2 GEMM

<a id="ime2"></a>

One A100 core. Each row is that model’s feed-forward gate shape, `K` = hidden size and `N` = intermediate size. Weights are packed once, then `Ime2HpGemm` is timed. GOP/s is `2·M·N·K / t`. DeepSeek R1 distill Qwen 1.5B has the same shape as Qwen2.5 1.5B.

| Model | K | N | M=1 | M=4 |
| ----- | -: | -: | ----: | ----: |
| SmolLM2 135M | 576 | 1536 | 28.18 | 69.37 |
| SmolLM2 360M | 960 | 2560 | 34.23 | 86.75 |
| Qwen2.5 0.5B | 896 | 4864 | 32.19 | 81.73 |
| TinyLlama 1.1B | 2048 | 5632 | 37.20 | 95.25 |
| Qwen2.5 1.5B | 1536 | 8960 | 37.05 | 95.06 |
| StableLM 2 1.6B | 2048 | 5632 | 37.04 | 94.77 |
| SmolLM2 1.7B | 2048 | 8192 | 37.06 | 94.07 |
| Phi-2 | 2560 | 10240 | 36.48 | 92.79 |
| Qwen2.5 3B | 2048 | 11008 | 36.96 | 94.41 |

## Install

Compile on the X100 cores (0–7). The login shell on this board is already pinned there. `/proc/cpuinfo` lists `zvfh` and the bf16 vector extensions and does not list a `zvl` length, so the RISC-V cmake check stops unless the length is passed. The RVV kernels in this tree handle 128 and 256.

```bash
git clone --depth 1 -b k3-a100-ime2 https://github.com/opensolvers/vllm.git
cd vllm
python3 -m venv "$HOME/vllm-venv" --system-site-packages
"$HOME/vllm-venv/bin/pip" install 'setuptools>=77,<81' setuptools-scm setuptools-rust ninja
"$HOME/vllm-venv/bin/pip" install torch==2.14.0 \
  --extra-index-url https://pypi.riseproject.dev/simple/
env VLLM_TARGET_DEVICE=cpu CMAKE_BUILD_TYPE=Release MAX_JOBS=3 \
  CMAKE_BUILD_PARALLEL_LEVEL=3 \
  CMAKE_ARGS="-DVLLM_RVV_VLEN=256 -DCMAKE_DISABLE_FIND_PACKAGE_CUDA=ON -DCMAKE_BUILD_TYPE=Release" \
  "$HOME/vllm-venv/bin/pip" install . --no-build-isolation --no-deps
"$HOME/vllm-venv/bin/pip" install -r requirements/cpu.txt
```

Torch comes from the [RISE wheel index](https://pypi.riseproject.dev/simple/). That wheel’s OpenBLAS has `sgemm_` and no `sbgemm_`. Defining `VLLM_HAS_OPENBLAS` anyway makes `_C` fail to load, so the branch runs `nm -D` and defines the macro only when `sbgemm_` is present. BF16 GEMM then uses the PyTorch BLAS path.

On riscv64, `requirements/cpu.txt` builds sentencepiece, tiktoken, llvmlite, numba, orjson, outlines_core, and opencv-python-headless from source. Numba imports. Its JIT aborts on an unimplemented RISC-V relocation (`R_RISCV_SET8`), so the N-gram speculative path is not usable. `xgrammar` is not built for this architecture, and the `vllm` command imports it, so the CLI does not start. `from vllm import LLM` does.

```python
from vllm import LLM, SamplingParams

llm = LLM(
    model="HuggingFaceTB/SmolLM2-135M-Instruct",
    dtype="bfloat16",
    max_model_len=192,
    enforce_eager=True,
)
out = llm.generate(
    ["The capital of France is"],
    SamplingParams(temperature=0, max_tokens=16),
)
```

## How the IME2 kernel is built

<a id="kernel"></a>

The CPU op is `torch.ops._C.ime2_hp_gemm_cpu`. `A` is float `[M, K]`, `qweight` is packed nibbles `uint8 [N, K/32, 16]`, `scales` is float `[N, K/32]`, and the result is float `[M, N]`. `K` is a multiple of 32.

The tile is the same Q4_0×32 layout as the ONNX kernel in [`sqnbitgemm_ime_hp.inc`](https://github.com/opensolvers/onnxruntime/blob/k3-a100-ime2/onnxruntime/core/mlas/lib/riscv64/sqnbitgemm_ime_hp.inc). Eight K32 subblocks share one fp16 superblock scale. The symmetric zero point 8 is folded into the activation row sum: the stored sum is `fp16(-sum) * 8`. A subblock is an fp16 scale plus 32 int8 values. One activation row per 256 K is 290 bytes. One B block (K32×N32) is 576 bytes: 32 fp16 scales, then the nibbles.

The inner loop is `vmadotsu.hp` and `vmadotu.hp` with `i4`, four N groups at a time. M is taken four rows at a time, then one. Columns past a multiple of 32 use the scalar tail

```text
acc += scale_avg * as * bs * (dot + asum)
```

where `dot` is the sum of int8 times nibble. On random tiles the assembly and that tail stay within about 0.7 absolute, which is the fp16 rounding of the scales. The Python op matches the standalone assembly.

The file is compiled with `-march=rv64gcv_zvfh_zfbfmin_zvfbfmin_zvl1024b -mrvv-vector-bits=zvl`. At `zvl256b`, GCC emits `vs1r.v` for a 32-byte register. On the A100 that instruction stores the whole 128-byte register and walks off the stack. The rest of `_C` stays at VLEN 256. The op reads `vlenb` and returns an error unless it is 128.

Loading `_C` while the process is already pinned to cores 8–15 aborts: the other objects were compiled for VLEN 256, and their vector spills assume that length. Import on cores 0–7, then move:

```python
import os
import torch
import vllm._C

with open("/proc/set_ai_thread", "w") as handle:
    handle.write(str(os.getpid()))
os.sched_setaffinity(0, set(range(8, 16)))
y = torch.ops._C.ime2_hp_gemm_cpu(a, qweight, scales)
```

See [SM10](../boards/SM10.html#getting-onto-a100-procset_ai_thread) for the core fence. The generation table packs each weight once and splits the N tiles across the eight A100 cores. The torch op still packs on every call. vLLM’s `int4_scaled_mm_cpu` path unpacks int4 and runs a VLEN-256 GEMM, and its weight layout is different.
