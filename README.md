# AI Study Notes 🤖

A personal knowledge base for AI research, model selection, and local LLM experiments.

---

## 📌 Chapter 1: AI Model Selection & Precision Formats

### Background

When running large language models (LLMs) locally, one of the most important decisions is choosing the right **model format and precision**. The same model can exist in multiple formats, each with different trade-offs in quality, speed, and VRAM usage.

This was first explored during setup of the **Asus Ascent GX10** (NVIDIA GB10 Blackwell, 121.6 GiB VRAM) running local models via **Ollama**.

---

### Precision Formats Explained

#### BF16 (Brain Float 16)
- **Bits:** 16-bit floating point (1 sign + 8 exponent + 7 mantissa)
- **Size (27B model):** ~54–56 GB
- **Quality:** Closest to original training quality — zero quantization loss
- **Speed:** Baseline (slowest among the three)
- **Use case:** When maximum quality is required and VRAM is abundant
- **Platform:** Universal (any GPU supporting BF16)

#### FP8 (Float Point 8)
- **Bits:** 8-bit floating point (E4M3 format: 1 sign + 4 exponent + 3 mantissa)
- **Size (27B model):** ~27 GB
- **Quality:** Extremely close to BF16 — less than 1% degradation
- **Speed:** ~1.5–2x faster than BF16 on supported hardware
- **Use case:** Best balance of quality and speed on modern NVIDIA GPUs
- **Platform:** **NVIDIA Hopper (H100) / Blackwell (GB10) native hardware support**
- **Framework:** HuggingFace / vLLM (not yet supported in Ollama)

#### Q8_0 (GGUF INT8 Quantization)
- **Bits:** 8-bit integer (not floating point — grouped quantization with scale factors)
- **Size (27B model):** ~30 GB
- **Quality:** Very close to BF16 — minimal degradation
- **Speed:** Fast; works on CPU+GPU hybrid or pure GPU
- **Use case:** Best general-purpose format for Ollama users
- **Platform:** **Universal — Linux, macOS, Windows** ✅
- **Framework:** llama.cpp / Ollama (fully supported)

---

### Key Differences at a Glance

| Format | Type | Size (27B) | Quality Loss | Platform | Ollama Support |
|--------|------|-----------|--------------|----------|----------------|
| BF16   | Float16 | ~54GB | None (baseline) | Universal | ✅ |
| FP8    | Float8  | ~27GB | <1% | NVIDIA H100/GB10 | ❌ (vLLM needed) |
| Q8_0   | INT8 GGUF | ~30GB | Minimal | Universal | ✅ |
| Q4_K_M | INT4 GGUF | ~17GB | Noticeable on complex tasks | Universal | ✅ |

#### a3b (Activated 3 Billion)
- Refers to **MoE (Mixture of Experts)** architecture where only ~3B parameters are **activated per token**, even though the total model size is larger (e.g., 35B)
- Example: `qwen3.6:35b-a3b` = 35B total params, but only 3B active during inference
- **Benefit:** Near-35B quality at ~3B compute cost — very fast and memory-efficient
- **Size:** ~24GB despite being a "35B" model

#### MLX
- Apple's **machine learning framework** for Apple Silicon (M1/M2/M3/M4 chips)
- Optimized for **unified memory architecture** on Mac
- Tags with `mlx` (e.g., `qwen3.6:27b-mlx-bf16`) are **macOS only** — will fail on Linux
- Equivalent to CUDA on NVIDIA, but for Apple

#### MXFP8 (Microscaling FP8)
- A newer variant of FP8 developed by **Microsoft + NVIDIA + AMD** (OCP MX spec)
- Uses **block-level scaling** (finer granularity than standard FP8) → better accuracy
- In Ollama context: tags like `qwen3.6:27b-mxfp8` are **macOS MLX format** despite the name
- On Linux with NVIDIA, true MXFP8 support requires PyTorch 2.5+ / vLLM

#### NVFP4 (NVIDIA Float Point 4)
- NVIDIA's proprietary **4-bit floating point** format for Blackwell GPUs (GB10, B100, B200)
- Uses FP4 with microscaling — much higher quality than standard INT4 quantization
- In Ollama: tags like `qwen3.6:27b-nvfp4` are **macOS MLX format** (misleading name!)
- Native NVFP4 on Linux requires NVIDIA TensorRT-LLM or vLLM with Blackwell support

---

> **Key insight:** Q8_0 and FP8 are both "8-bit" but fundamentally different technologies. Q8_0 uses integer quantization (GGUF format), while FP8 uses native floating-point hardware acceleration. For Ollama users, Q8_0 is the practical equivalent.

---

### Available qwen3.6 Tags on Ollama

Tested on: [https://ollama.com/library/qwen3.6/tags](https://ollama.com/library/qwen3.6/tags)

| Tag | Size | Format | Linux+NVIDIA |
|-----|------|--------|-------------|
| `qwen3.6:27b` | 17GB | Q4_K_M | ✅ |
| `qwen3.6:27b-q8_0` | 30GB | Q8_0 | ✅ |
| `qwen3.6:27b-bf16` | 56GB | BF16 | ✅ |
| `qwen3.6:35b-a3b` | 24GB | Q4_K_M | ✅ |
| `qwen3.6:35b-a3b-q8_0` | 39GB | Q8_0 | ✅ |
| `qwen3.6:35b-a3b-bf16` | 71GB | BF16 | ✅ |
| `qwen3.6:27b-mxfp8` | 31GB | MXFP8 | ❌ macOS only |
| `qwen3.6:27b-nvfp4` | 20GB | NVFP4 | ❌ macOS only |

> ⚠️ Despite names like `nvfp4` and `mxfp8` sounding NVIDIA-specific, these tags are **macOS MLX format only** and will return a 412 error on Linux.

---

### First Model Selected for Testing

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

## 🗂️ Index

- [Chapter 1: Model Selection & Precision Formats](#-chapter-1-ai-model-selection--precision-formats) ✅
- Chapter 2: Benchmark Framework *(coming soon)*
- Chapter 3: vLLM vs Ollama Comparison *(coming soon)*
- Chapter 4: Model Quality Evaluation *(coming soon)*

---

*Started: 2026-05-20 | Hardware: Asus Ascent GX10 / NVIDIA GB10 / 121.6 GiB VRAM*
