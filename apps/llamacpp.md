---
title: llama.cpp on RISC-V — X60 IME vs RVV
description: End-to-end Q4_0 GGUF validation on Orange Pi RV2 — ten models, IME smt.vmadot prefill vs RVV token-gen, coherence and llama-bench results.
---

# llama.cpp

[llama.cpp](https://github.com/ggml-org/llama.cpp) Q4_0 inference on the SpaceMiT X60 — end-to-end validation of the **IME `smt.vmadot`** path against a plain **RVV** build on the Orange Pi RV2.

Benchmark source: [opensolvers/benchmarks/llamacpp](https://github.com/opensolvers/benchmarks/tree/main/llamacpp) — model list, `validate_model.sh` / `run_all.sh`, and `model_validation.tsv`. Kernel-level IME work: [IME on RV2](../boards/RV2.html#ime-integer-matrix-extension) / [benchmarks/ime](https://github.com/opensolvers/benchmarks/tree/main/ime). Related ORT int4 path: [ONNX Runtime](onnx.html).

## What it probes

Where [`ime/`](https://github.com/opensolvers/benchmarks/tree/main/ime) isolates the `smt.vmadot` microkernel, this suite asks: **which real Q4_0 models fit and run, and where does IME actually help?**

- **IME build** — SpaceMiT IME1 backend (`use_ime1: 1` in the startup log)
- **RVV build** — plain vector control arm (`~/x60-rvv`)
- Bench: `llama-bench -p 128 -n 32 -t 8` (pp128 / tg32)
- Coherence: `llama-cli` prompt; pass = loads, finite coherent output, no repetition-collapse

## Board ceiling

7.7 GiB RAM, **no swap** → ~5.5 GiB safe model+KV budget. Largest safe fit is **7–7.6B at Q4_0**. ≥13B Q4_0 and 7–8B at Q8_0/fp16 will OOM.

IME accelerates **Q4_0** specifically; other quants run but on the RVV/scalar path.

## Building the IME backend (toolchain)

`smt.vmadot` has **no compiler intrinsics**. Mainline support is assembler-only: **LLVM ≥ 22**, **GCC ≥ 16**, **binutils ≥ 2.46** (`xsmtvdot`). EESSI `foss-2025b` is still GCC **14.3** + binutils **2.44**, so the IME llama.cpp build needs:

1. **Source patch** [`llama.cpp-x60-ime-upstream-binutils.patch`](https://github.com/opensolvers/benchmarks/blob/main/ime/llama.cpp-x60-ime-upstream-binutils.patch) — rename ggml's vendor `vmadot` → upstream **`smt.vmadot`**, inject `.option arch,+xsmtvdot` in the inline asm, and skip IME2 sources (X60 is IME1-only).
2. **Patched assembler** — deploy standalone **`binutils-2.46.1-xsmtvdot`** and hand GCC **only that `as`** via `-B` (private symlink dir) so `FindSMTIME` and every compile see the mnemonic while linking keeps the toolchain `ld`.

Details and the raw-`.insn` alternative used by `ime-bench`: [RV2 IME — toolchain support](../boards/RV2.html#toolchain-support-xsmtvdot).

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

## Kernel optimization (scale-build)

Isolated IME1 Q4_0 GEMM patch: **+4.3%** pp512 microkernel (22.50 → 23.47 GOP/s), bit-exact — see [RV2 IME scale-build](../boards/RV2.html#ime1-scale-build-prefill-optimization-llamacpp). End-to-end `llama-bench` pp512 on this board sits under ±15–20% noise; decode (`tg`) uses the untouched M1/GEMV path.

## Reproduce

Requires IME and RVV builds on the board (`~/x60-ime`, `~/x60-rvv`) plus a models directory.

```bash
# single model
./validate_model.sh \
  "https://huggingface.co/Qwen/Qwen2.5-1.5B-Instruct-GGUF/resolve/main/qwen2.5-1.5b-instruct-q4_0.gguf" \
  qwen2.5-1.5b.q4_0.gguf qwen2.5-1.5b 1.5B

# full sweep (all 10)
./run_all.sh
```

Each run downloads the Q4_0 GGUF, coherence-checks on IME, benches IME then RVV, appends a TSV row, and deletes the GGUF to respect the no-swap RAM ceiling.

## Takeaways

1. **IME wins prefill ≥1.1B** (1.2–2.6×); RVV wins token-gen everywhere.
2. **Pick the build by workload** — batch/prefill vs interactive decode.
3. **7B Q4_0 is the practical ceiling** on 8 GB no-swap; keep ctx modest on ≥3B.
4. **Kernel + models** — microkernel A/B in `ime/`; coverage here in `llamacpp/`.
