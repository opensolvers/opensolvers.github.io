---
title: llama.cpp on RISC-V — X60 IME vs RVV
description: End-to-end llama.cpp on Orange Pi RV2 — Q4_0 IME vs RVV, Q4_K_M m1gemv study, and opensolvers/llama.cpp x60-ime-rvv kernel work.
---

# llama.cpp

[llama.cpp](https://github.com/ggml-org/llama.cpp) Q4_0 / Q8_0 inference on the SpaceMiT X60 — end-to-end validation of the **IME `smt.vmadot`** path against a plain **RVV** build on the Orange Pi RV2.

Benchmark source: [opensolvers/benchmarks/llamacpp](https://github.com/opensolvers/benchmarks/tree/main/llamacpp) — model list, `validate_model.sh` / `run_all.sh`, and `model_validation.tsv`. Kernel-level IME work: [IME on RV2](../boards/RV2.html#ime-integer-matrix-extension) / [benchmarks/ime](https://github.com/opensolvers/benchmarks/tree/main/ime). Related ORT int4 path: [ONNX Runtime](onnx.html).

## Our fork: `opensolvers/llama.cpp` (`x60-ime-rvv`)

Upstream [ggml-org/llama.cpp](https://github.com/ggml-org/llama.cpp) already ships a SpaceMiT backend (`GGML_CPU_RISCV64_SPACEMIT`). We keep a **fork** — [opensolvers/llama.cpp](https://github.com/opensolvers/llama.cpp) — and stage X60-specific work on the **[`x60-ime-rvv`](https://github.com/opensolvers/llama.cpp/tree/x60-ime-rvv)** branch so we can iterate on real silicon without blocking upstream review.

**Why a fork / branch (not patches-only):**

- Core-level kernels (IME GEMM, RVV softmax / GEMV) change often; a branch is the durable place to stack, bisect, and A/B them.
- Some changes are **IME1 / X60-shaped** (cluster-0 matrix unit, `xsmtvdot` mnemonics, IME2 gated out) and need a landing pad before they are ready for `master`.
- Viewers can **clone one branch** and get the same software we measure on the RV2 — no hunting across EasyBuild patch files.

**What is on `x60-ime-rvv` today** (stacked on upstream master):

| # | Change | Status | Measured impact |
| - | ------ | ------ | --------------- |
| [#1](https://github.com/opensolvers/llama.cpp/pull/1) | IME1 i8i8 GEMM for **Q8_0** / **Q6_K** `MUL_MAT` | merged | Prefill path via `smt.vmadot`; repack opt-in so default tg stays RVV-friendly |
| [#2](https://github.com/opensolvers/llama.cpp/pull/2) | IME1 **scale-build** (`LOAD_SCALE_4x16_FP16_OPT`) | merged | **+4.3%** isolated Q4_0 GEMM (22.50 → 23.47 GOP/s), bit-exact — [RV2](../boards/RV2.html#ime1-scale-build-prefill-optimization-llamacpp) |
| [#3](https://github.com/opensolvers/llama.cpp/pull/3) | Upstream **`xsmtvdot` / `smt.vmadot`** asm + IME2 gate | merged | Builds against binutils ≥ 2.46; X60 skips IME2 sources |
| [#4](https://github.com/opensolvers/llama.cpp/pull/4) | RVV **SOFT_MAX** F32 | merged | **212/212** `test-backend-ops` OK; ~**2.1×** vs scalar on `[4096,4096,5]` |
| [#5](https://github.com/opensolvers/llama.cpp/pull/5) | RVV **M1 int8 GEMV** (token-gen / remainder rows) | open | Bit-exact; standalone **9.36×** vs scalar; tg64 on Qwen2.5-0.5B **Q8_0**: **0.82 → 5.29 t/s (6.45×)** — but see [Q4_K_M caveat](#study-2--q4_k_m-17-models-m1gemv-vs-rvv) |

Upstream already ships the base SpaceMiT Q4_0 IME path; the fork stacks the kernels above on top of that.

### How to use it

On a SpaceMiT X60 board (Orange Pi RV2 / Banana Pi F3), with a toolchain that can assemble `smt.vmadot` ([details](../boards/RV2.html#toolchain-support-xsmtvdot)):

```bash
git clone -b x60-ime-rvv https://github.com/opensolvers/llama.cpp.git
cd llama.cpp

cmake -B build -DCMAKE_BUILD_TYPE=Release \
  -DGGML_CPU_RISCV64_SPACEMIT=ON \
  -DGGML_RVV=ON -DGGML_RV_ZVFH=ON -DGGML_RV_ZFH=ON \
  -DGGML_RV_ZBA=ON -DGGML_RV_ZICBOP=ON
cmake --build build --parallel "$(nproc)"

# Prefer Q4_0 for the IME Q4 path; pin to cluster 0 for IME (cores 0–3)
./build/bin/llama-cli -m model.q4_0.gguf -t 4 -c 2048 -n 64 \
  -p "Explain RISC-V vector extensions in two sentences."
# Startup should report use_ime1: 1

./build/bin/llama-bench -m model.q4_0.gguf -p 128 -n 32 -t 4
```

- **Docs on the branch:** [docs/build-riscv64-spacemit.md](https://github.com/opensolvers/llama.cpp/blob/x60-ime-rvv/docs/build-riscv64-spacemit.md) (SpaceMiT cross-toolchain / QEMU notes).
- **Validation suite we publish numbers from:** [benchmarks/llamacpp](https://github.com/opensolvers/benchmarks/tree/main/llamacpp) (`validate_model.sh` / `run_all.sh`).
- For A/B against plain RVV, build once **with** `GGML_CPU_RISCV64_SPACEMIT=ON` and once **without** (or use two install prefixes) — same pattern as the tables below.

## What it probes

Where [`ime/`](https://github.com/opensolvers/benchmarks/tree/main/ime) isolates the `smt.vmadot` microkernel, this suite asks: **which real Q4_0 models fit and run, and where does IME actually help?**

- **IME build** — SpaceMiT IME1 backend (`use_ime1: 1` in the startup log)
- **RVV build** — plain vector control arm (`~/x60-rvv`)
- Bench: `llama-bench -p 128 -n 32 -t 8` (pp128 / tg32)
- Coherence: `llama-cli` prompt; pass = loads, finite coherent output, no repetition-collapse

## Board ceiling

7.7 GiB RAM, **no swap** → ~5.5 GiB safe model+KV budget. Largest safe fit is **7–7.6B at Q4_0**. ≥13B Q4_0 and 7–8B at Q8_0/fp16 will OOM.

IME accelerates **Q4_0** (and Q8_0 / Q6_K via the fork GEMM path). **Q4_K_M** does not engage `smt.vmadot` today — use the RVV build (see [study 2](#study-2--q4_k_m-17-models-m1gemv-vs-rvv)).

## Building the IME backend (toolchain)

`smt.vmadot` has **no compiler intrinsics**. Mainline support is assembler-only: **LLVM ≥ 22**, **GCC ≥ 16**, **binutils ≥ 2.46** (`xsmtvdot`). EESSI `foss-2025b` is still GCC **14.3** + binutils **2.44**.

On the [`x60-ime-rvv`](https://github.com/opensolvers/llama.cpp/tree/x60-ime-rvv) fork branch those asm/CMake fixes are **already applied**. You still need an assembler that understands `smt.vmadot` — typically a standalone **`binutils-2.46+xsmtvdot`** pointed at GCC via `-B` only for `as` (so linking keeps the toolchain `ld`). Older EasyBuild patches that did the same rename live under [benchmarks/ime](https://github.com/opensolvers/benchmarks/tree/main/ime) for reference.

Raw-`.insn` alternative (no special assembler): [RV2 IME — toolchain support](../boards/RV2.html#toolchain-support-xsmtvdot).

## Headline results (10/10 validated)

Every model **loaded, reported `use_ime1: 1`, and produced coherent finite output.** No OOM.

| Phase | Winner | Pattern |
| ----- | ------ | ------- |
| **Prefill (pp)** | **IME** on every model ≥1.1B | up to ~2.5× RVV (Mistral-7B: 5.84 vs 2.35 t/s); only 0.5B favours RVV (IME setup overhead) |
| **Token-gen (tg)** | **RVV** universally | ~5–8× at small sizes → ~1.9× at 7B; IME tg is roughly half RVV throughout |

**Net:** prompt-heavy / batch → IME build; generation-heavy / interactive → RVV build.

tg scales down with model size as the ~3.85 GB/s memory-bandwidth wall predicts (RVV: 12.18 → 1.38 tg t/s from 0.5B → 7B).

## Measured — IME vs RVV (Orange Pi RV2)

| Model | Params | Q4_0 (MiB) | pp128 IME | tg32 IME | pp128 RVV | tg32 RVV | pp win | tg win |
| ----- | -----: | ---------: | --------: | -------: | --------: | -------: | :----: | :----: |
| Qwen2.5-0.5B-Instruct | 0.5B | 408 | 69.03 | 1.47 | 100.60 | 12.18 | RVV | RVV |
| TinyLlama-1.1B-Chat | 1.1B | 608 | 37.72 | 1.72 | 34.95 | 9.47 | **IME** | RVV |
| Llama-3.2-1B-Instruct | 1.2B | 737 | 38.87 | 2.00 | 25.27 | 7.74 | **IME** | RVV |
| Qwen2.5-1.5B-Instruct | 1.5B | 1016 | 27.14 | 1.19 | 22.77 | 6.10 | **IME** | RVV |
| Gemma-2-2B-it | 2.6B | 1554 | 17.56 | 1.01 | 10.04 | 3.15 | **IME** | RVV |
| Qwen2.5-3B-Instruct | 3.1B | 1905 | 14.13 | 0.85 | 7.85 | 3.12 | **IME** | RVV |
| Llama-3.2-3B-Instruct | 3.2B | 1832 | 13.99 | 1.05 | 6.95 | 3.09 | **IME** | RVV |
| Phi-3.5-mini-instruct | 3.8B | 2081 | 11.37 | 1.11 | 4.59 | 2.02 | **IME** | RVV |
| Mistral-7B-Instruct-v0.3 | 7.2B | 3922 | 5.84 | 0.73 | 2.35 | 1.38 | **IME** | RVV |
| Qwen2.5-7B-Instruct | 7.6B | 3798 | 6.06 | 0.73 | 2.52 | 1.38 | **IME** | RVV |

Raw TSV: [`model_validation.tsv`](https://github.com/opensolvers/benchmarks/blob/main/llamacpp/model_validation.tsv). Full model notes and HF sources: [`models.md`](https://github.com/opensolvers/benchmarks/blob/main/llamacpp/models.md).

## Optimizations in detail

### IME1 Q8_0 / Q6_K GEMM ([#1](https://github.com/opensolvers/llama.cpp/pull/1))

Adds an IME1 `smt.vmadot` int8 GEMM fast path for **Q8_0** and **Q6_K** `MUL_MAT` / `MUL_MAT_ID`. The i8i8 repack is **opt-in / prefill-oriented**, so default token-generation does not pay for a layout that only helps large-M GEMM. Verified on-board: **64 `smt.vmadot`** instructions in `libggml-cpu.so` under xsmtvdot-aware objdump.

### IME1 scale-build ([#2](https://github.com/opensolvers/llama.cpp/pull/2))

Replaces the masked `vfmul.vf` scale-combine chain with `LOAD_SCALE_4x16_FP16_OPT` (`vfmul.vv`). Isolated interleaved A/B on RV2: **+4.3%** pp512 (22.50 → 23.47 GOP/s), bit-exact, 30/30 rounds. End-to-end `llama-bench` pp512 on this multi-tenant board sits under ±15–20% noise. Details: [RV2 IME scale-build](../boards/RV2.html#ime1-scale-build-prefill-optimization-llamacpp).

### `xsmtvdot` binutils alignment ([#3](https://github.com/opensolvers/llama.cpp/pull/3))

Inline asm emits `.option arch,+xsmtvdot` + `smt.vmadot` (encoding `0xe241b12b`); CMake only builds IME2 when `xsmtvdotii` is present. Lets EESSI GCC 14 + a standalone binutils 2.46 `as` build the backend — [toolchain notes](../boards/RV2.html#toolchain-support-xsmtvdot).

### RVV SOFT_MAX ([#4](https://github.com/opensolvers/llama.cpp/pull/4))

Vectorized F32 `SOFT_MAX` in the SpaceMiT backend (mask / sinks / ALiBi). Correctness: **212/212** `test-backend-ops`. Perf vs scalar (~0.98 GB/s baseline):

| Shape `ne` | GB/s | Speedup |
| ---------- | ---: | ------: |
| **[4096,4096,5]** | **2.09** | **~2.1×** |
| [1024,1024,10] | 2.15 | ~2.2× |
| [64,64,20] | 5.15 | ~5.3× |
| large single-row (≥65536) | 1.08–1.35 | ~1.1–1.4× (bandwidth-bound) |

### RVV M1 int8 GEMV ([#5](https://github.com/opensolvers/llama.cpp/pull/5), open)

Vectorizes the scalar M=1 / remainder-row i8i8 path used in token-gen (`vlse8` + widening `vwmacc` + fused `vfmacc` for bit-exact scales). On **Q8_0**, tg64 for Qwen2.5-0.5B jumps **0.82 → 5.29 t/s (6.45×)**. On **Q4_K_M**, the IME `smt.vmadot` fast path does **not** engage — the m1gemv build is then a **net regression** vs plain RVV (next section).

## Study 2 — Q4_K_M, 17 models (`m1gemv` vs RVV)

Companion to the Q4_0 table above: **17 models** (0.5B–4B), **Q4_K_M**, `llama-bench -p 512 -n 64 -t 8`, IME build with the M1-GEMV-tuned kernel vs the same `~/x60-rvv` control. Source: [results_17model_m1gemv_vs_rvv.md](https://github.com/opensolvers/benchmarks/blob/main/llamacpp/results_17model_m1gemv_vs_rvv.md).

**Finding:** on K-quants, **`m1gemv` is a net regression**.

- **tg64:** RVV wins **17/17**, by **2.9–6.5×** (e.g. Qwen2.5-0.5B: 1.42 IME vs 9.19 RVV).
- **pp512:** near parity; RVV ahead on most models. IME wins pp on only three (qwen2.5-0.5b, phi-3.5-mini, phi-4-mini).

Why: Q4_K_M's 256-element superblocks / 6-bit scales do **not** map onto the IME `smt.vmadot` tiling that won prefill on Q4_0, so the kernel falls back and still pays m1gemv overhead on decode.

| label | GB | pp512 IME | pp512 RVV | tg64 IME | tg64 RVV |
| ----- | -: | --------: | --------: | -------: | -------: |
| qwen2.5-0.5b | 0.37 | 23.92 | 20.73 | 1.42 | **9.19** |
| llama-3.2-1b | 0.75 | 24.13 | **27.49** | 1.90 | **8.19** |
| gemma-3-1b | 0.75 | **12.75** | 11.36 | 1.04 | **4.40** |
| deepseek-coder-1.3b | 0.81 | 16.51 | 16.43 | 1.37 | **6.91** |
| qwen2.5-1.5b | 0.92 | 18.62 | 18.80 | 1.14 | **6.31** |
| stablelm-2-1.6b | 0.96 | 16.58 | **20.65** | 1.17 | **6.74** |
| falcon3-1b | 0.98 | 19.57 | **22.49** | 1.69 | **7.28** |
| smollm2-1.7b | 0.98 | 13.62 | **15.46** | 1.35 | **5.94** |
| deepseek-r1-qwen-1.5b | 1.04 | 18.54 | 19.10 | 1.14 | **6.38** |
| granite-3.1-2b | 1.44 | 9.23 | **10.48** | 0.80 | **4.04** |
| gemma-2-2b | 1.59 | 11.43 | 12.07 | 0.97 | **3.25** |
| qwen2.5-3b | 1.80 | 8.25 | 8.93 | 0.79 | **3.43** |
| falcon3-3b | 1.87 | 10.37 | 10.68 | 1.17 | **3.74** |
| llama-3.2-3b | 1.88 | 8.59 | 9.14 | 0.96 | **3.30** |
| phi-3.5-mini | 2.23 | **4.08** | 2.91 | 0.69 | **2.04** |
| gemma-3-4b | 2.32 | 7.67 | 7.68 | 0.70 | **2.52** |
| phi-4-mini | 2.32 | **5.33** | 4.14 | 0.76 | **2.17** |

**Default to RVV for Q4_K_M.** Keep the IME build for **Q4_0** (and Q8_0 once #5 lands) where `smt.vmadot` actually fires. Next kernel target: engage IME on K-quant superblocks, or gate m1gemv off for K-quants.

## Reproduce

### Study 1 (Q4_0, 10 models)

Requires IME and RVV builds on the board (`~/x60-ime`, `~/x60-rvv`) plus a models directory.

```bash
./validate_model.sh \
  "https://huggingface.co/Qwen/Qwen2.5-1.5B-Instruct-GGUF/resolve/main/qwen2.5-1.5b-instruct-q4_0.gguf" \
  qwen2.5-1.5b.q4_0.gguf qwen2.5-1.5b 1.5B

./run_all.sh
```

### Study 2 (Q4_K_M, 17 models)

```bash
./bench_suite.sh       # m1gemv IME arm  → results.md
./bench_suite_rvv.sh   # RVV control     → results_rvv.md
```

Harnesses and TSVs: [benchmarks/llamacpp](https://github.com/opensolvers/benchmarks/tree/main/llamacpp).

## Takeaways

1. **Q4_0:** IME wins prefill ≥1.1B (1.2–2.6×); RVV wins token-gen everywhere.
2. **Q4_K_M:** plain RVV wins tg (and usually pp); m1gemv IME regresses until K-quants hit `smt.vmadot`.
3. **Fork kernels:** Q8_0/Q6_K IME GEMM, +4.3% scale-build, RVV softmax (~2×), M1 GEMV (**6.45×** tg on Q8_0 — [#5](https://github.com/opensolvers/llama.cpp/pull/5)).
4. **7B Q4_0 is the practical ceiling** on 8 GB no-swap; keep ctx modest on ≥3B.
5. **Pick build by quant + workload** — not “IME always” or “RVV always”.