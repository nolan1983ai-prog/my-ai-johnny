# Note 001 — AI Model Precision Formats & Ollama Model Selection

**Date:** 2026-05-20
**Hardware:** Asus Ascent GX10 / NVIDIA GB10 Blackwell / 121.6 GiB VRAM
**Framework:** Ollama v0.24.0

---

## Precision Formats Explained

When running large language models (LLMs) locally, one of the most important decisions is choosing the right **model format and precision**. The same model can exist in multiple formats, each with different trade-offs in quality, speed, and VRAM usage.

### BF16 (Brain Float 16)
- **Bits:** 16-bit floating point (1 sign + 8 exponent + 7 mantissa)
- **Size (27B model):** ~54–56 GB
- **Quality:** Closest to original training quality — zero quantization loss
- **Speed:** Baseline (slowest among the three)
- **Use case:** When maximum quality is required and VRAM is abundant
- **Platform:** Universal (any GPU supporting BF16)

### FP8 (Float Point 8)
- **Bits:** 8-bit floating point (E4M3 format: 1 sign + 4 exponent + 3 mantissa)
- **Size (27B model):** ~27 GB
- **Quality:** Extremely close to BF16 — less than 1% degradation
- **Speed:** ~1.5–2x faster than BF16 on supported hardware
- **Use case:** Best balance of quality and speed on modern NVIDIA GPUs
- **Platform:** NVIDIA Hopper (H100) / Blackwell (GB10) native hardware support
- **Framework:** HuggingFace / vLLM (not yet supported in Ollama)

### Q8_0 (GGUF INT8 Quantization)
- **Bits:** 8-bit integer (grouped quantization with scale factors — not floating point)
- **Size (27B model):** ~30 GB
- **Quality:** Very close to BF16 — minimal degradation
- **Speed:** Fast; works on CPU+GPU hybrid or pure GPU
- **Use case:** Best general-purpose format for Ollama users
- **Platform:** Universal — Linux, macOS, Windows ✅
- **Framework:** llama.cpp / Ollama (fully supported)

---

## Terminology

### a3b (Activated 3 Billion)
- Refers to **MoE (Mixture of Experts)** architecture where only ~3B parameters are **activated per token**, even though the total model size is larger (e.g., 35B)
- Example: `qwen3.6:35b-a3b` = 35B total params, but only 3B active during inference
- **Benefit:** Near-35B quality at ~3B compute cost — very fast and memory-efficient
- **Size:** ~24GB despite being a "35B" model

### MLX
- Apple's **machine learning framework** for Apple Silicon (M1/M2/M3/M4 chips)
- Optimized for **unified memory architecture** on Mac
- Tags with `mlx` (e.g., `qwen3.6:27b-mlx-bf16`) are **macOS only** — will fail on Linux
- Equivalent to CUDA on NVIDIA, but for Apple

### MXFP8 (Microscaling FP8)
- A newer variant of FP8 developed by **Microsoft + NVIDIA + AMD** (OCP MX spec)
- Uses **block-level scaling** (finer granularity than standard FP8) → better accuracy
- In Ollama context: tags like `qwen3.6:27b-mxfp8` are **macOS MLX format** despite the name
- On Linux with NVIDIA, true MXFP8 support requires PyTorch 2.5+ / vLLM

### NVFP4 (NVIDIA Float Point 4)
- NVIDIA's proprietary **4-bit floating point** format for Blackwell GPUs (GB10, B100, B200)
- Uses FP4 with microscaling — much higher quality than standard INT4 quantization
- In Ollama: tags like `qwen3.6:27b-nvfp4` are **macOS MLX format** (misleading name!)
- Native NVFP4 on Linux requires NVIDIA TensorRT-LLM or vLLM with Blackwell support

### e2b / e4b (Effective 2B / Effective 4B)
- The **"E" stands for "Effective" parameters** — used in Google's Gemma 4 edge models
- `e2b` = 2.3B effective parameters (but 5.1B total including embeddings)
- `e4b` = 4.5B effective parameters (but 8B total including embeddings)
- Designed specifically for **edge device deployments** (laptops, mobile)
- Support **Text + Image + Audio** input — multimodal at tiny size
- 128K context window
- Example: `ollama run gemma4:e2b` or `ollama run gemma4:e4b`
- Different from `a3b` (MoE active params) — e2b/e4b are dense models optimized for on-device use

### CUDA (Compute Unified Device Architecture)
- NVIDIA's **parallel computing platform and programming model** — the foundation of GPU-accelerated AI
- Launched in 2006; allows software to directly use NVIDIA GPU cores for general computation
- Every NVIDIA AI framework (PyTorch, TensorFlow, vLLM, Ollama) runs on top of CUDA
- **CUDA version** determines which GPU features and precision formats are available (e.g., FP8 requires CUDA 11.8+, NVFP4 requires CUDA 12.8+)
- Check version: `nvidia-smi` (top-right) or `nvcc --version`
- GX10 (GB10 Blackwell) ships with **CUDA 13.0** — supports all modern formats including FP8, MXFP8, NVFP4

---

## Quick Comparison Table

| Format | Type | Size (27B) | Quality Loss | Platform | Ollama Support |
|--------|------|-----------|--------------|----------|----------------|
| BF16   | Float16 | ~54GB | None (baseline) | Universal | ✅ |
| FP8    | Float8  | ~27GB | <1% | NVIDIA H100/GB10 | ❌ (vLLM needed) |
| Q8_0   | INT8 GGUF | ~30GB | Minimal | Universal | ✅ |
| Q4_K_M | INT4 GGUF | ~17GB | Noticeable on complex tasks | Universal | ✅ |

> ⚠️ **Q8_0 vs FP8:** Both are "8-bit" but fundamentally different. Q8_0 uses integer quantization (GGUF/llama.cpp), while FP8 uses native floating-point hardware. For Ollama users, Q8_0 is the practical equivalent of FP8.

---

## Available qwen3.6 Tags on Ollama

Reference: [https://ollama.com/library/qwen3.6/tags](https://ollama.com/library/qwen3.6/tags)

| Tag | Size | Format | Linux+NVIDIA |
|-----|------|--------|-------------|
| `qwen3.6:27b` | 17GB | Q4_K_M | ✅ |
| `qwen3.6:27b-q8_0` | 30GB | Q8_0 | ✅ |
| `qwen3.6:27b-bf16` | 56GB | BF16 | ✅ |
| `qwen3.6:35b-a3b` | 24GB | Q4_K_M (MoE) | ✅ |
| `qwen3.6:35b-a3b-q8_0` | 39GB | Q8_0 (MoE) | ✅ |
| `qwen3.6:35b-a3b-bf16` | 71GB | BF16 (MoE) | ✅ |
| `qwen3.6:27b-mxfp8` | 31GB | MXFP8 | ❌ macOS only |
| `qwen3.6:27b-nvfp4` | 20GB | NVFP4 | ❌ macOS only |
| `qwen3.6:27b-mlx-bf16` | 55GB | MLX BF16 | ❌ macOS only |

> ⚠️ Despite names like `nvfp4` and `mxfp8` sounding NVIDIA-specific, these tags are **macOS MLX format only** and will return HTTP 412 error on Linux.

---

## First Model Selected for Testing

**`qwen3.6:27b-q8_0`** — chosen as the benchmark starting point:

- Best quality available in Ollama on Linux + NVIDIA
- 30GB fits comfortably within GX10's 121.6 GiB VRAM
- Q8_0 is the practical "FP8 equivalent" within the Ollama/GGUF ecosystem
- Supports Vision + Text input, 256K context window

```bash
ollama pull qwen3.6:27b-q8_0
ollama run qwen3.6:27b-q8_0 "Hello! Introduce yourself."
```

---

## Supplementary: How MoE Works — The Router & 128 Experts

### Background
When we say `qwen3.6:35b-a3b` activates only 3B parameters per token, a natural question arises: **how does the model decide which 3B to use?**

---

### The Router Network

Every MoE layer contains a lightweight **Router Network (門控網絡)** whose sole job is:

> *"For this token, which Experts should handle it?"*

```
Input Token
    ↓
[Router Network]  ← lightweight neural net, learned during training
    ↓
Calculates a score for every Expert
    ↓
Selects Top-K highest scoring Experts (e.g. Top-8 out of 128)
    ↓
Only those Experts perform computation
    ↓
Results are weighted and merged → Output
```

---

### How the Router Learns to Choose

The Router is **trained jointly with the model** — it learns automatically from data:

- Math problems → activates Experts good at logical reasoning
- Chinese text → activates language-specific Experts
- Coding tasks → activates programming logic Experts
- General knowledge → activates knowledge-dense Experts

No human labels these specializations. The model **self-organizes** during training (emergent specialization).

---

### Are the 128 Experts Pre-defined?

**Yes — all 128 Experts are fixed in the model after training.**

Each Expert is a set of **FFN (Feed-Forward Network) weight matrices**, baked into the model file:

```
Layer 1: [Expert_001] [Expert_002] ... [Expert_128]
Layer 2: [Expert_001] [Expert_002] ... [Expert_128]
...
Layer N: [Expert_001] [Expert_002] ... [Expert_128]
```

Each layer has its own independent set of 128 Experts. They are all loaded into VRAM when the model starts.

**BUT** — what each Expert "specializes in" is implicit:
- ✅ **Structurally listed** — 128 fixed slots, all loaded into VRAM at runtime
- ❌ **No human-assigned labels** — nobody wrote "Expert 42 handles math"
- Researchers discover specializations after-the-fact via **interpretability analysis**

---

### The VRAM Paradox Explained

```
At inference time:
- All 128 Experts' parameters are loaded into VRAM  ✅  (why it needs 24GB)
- Per token: only 8 Experts actually compute           (why it's fast as 3B)
- Remaining 120 Experts: data present, no computation
```

This is why `35b-a3b` needs ~24GB VRAM (full model loaded) but runs at ~3B speed.

---

### Qwen3.6 35b-a3b Specific Numbers

| Property | Value |
|----------|-------|
| Total Experts per layer | 128 + 1 shared (always active) |
| Activated per token | Top-8 |
| Active parameters | ~3B |
| Total parameters | 35B |
| Router overhead | Negligible |

---

### Why 3B Active and Not More?

This is an **optimized balance found during training research**:
- Too few active experts → poor quality
- Too many active experts → loses efficiency advantage of MoE
- 3–4B active parameters is the current **sweet spot** for quality vs. speed

---

### Human Analogy

Imagine a company with 128 specialist employees:

1. Manager (Router) receives a task
2. Manager judges: this needs "Finance" + "Legal" + ... (8 people)
3. Only those 8 attend the meeting; other 120 rest
4. Their outputs are combined into one answer

Every token = a new task. The manager decides fresh each time. Nobody told the manager in advance who is the "Finance expert" — it figured that out through years of experience (training).

---

## Available gemma4 Tags on Ollama

Reference: [https://ollama.com/library/gemma4](https://ollama.com/library/gemma4)

### Model Variants Overview

| Tag | Size | Type | Active Params | Modalities | Context | Linux+NVIDIA |
|-----|------|------|--------------|-----------|---------|-------------|
| `gemma4:e2b` | ~5GB | Dense (Edge) | 2.3B effective | Text, Image, Audio | 128K | ✅ |
| `gemma4:e4b` | ~8GB | Dense (Edge) | 4.5B effective | Text, Image, Audio | 128K | ✅ |
| `gemma4:26b` | ~26GB | MoE | 3.8B active / 25.2B total | Text, Image | 256K | ✅ |
| `gemma4:31b` | ~31GB | Dense | 30.7B | Text, Image | 256K | ✅ |
| `gemma4:31b-cloud` | — | Cloud | — | Text, Image | 256K | ☁️ Ollama Cloud only |

### Benchmark Highlights (Gemma 4 family)

| Benchmark | 31B Dense | 26B MoE (A4B) | E4B | E2B |
|-----------|-----------|----------------|-----|-----|
| MMLU Pro | 85.2% | 82.6% | 69.4% | 60.0% |
| AIME 2026 | 89.2% | 88.3% | 42.5% | 37.5% |
| LiveCodeBench v6 | 80.0% | 77.1% | 52.0% | 44.0% |
| GPQA Diamond | 84.3% | 82.3% | 58.6% | 43.4% |

> **Key insight:** `gemma4:26b` (MoE) achieves near-31B quality at only 3.8B active parameters — similar concept to `qwen3.6:35b-a3b`. Great value for VRAM.

---

*← [Back to Index](../README.md)*
