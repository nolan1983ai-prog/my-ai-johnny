# Note 004 — Ollama Inference Parameters: A Deep Dive

**Date:** 2026-05-21
**Hardware:** Asus Ascent GX10 / NVIDIA GB10 Blackwell / 121.6 GiB VRAM
**Context:** Parameters available in Ollama's `options` block for controlling LLM inference behaviour

---

## Overview

When you send a request to Ollama, you can pass an `options` object to control how the model generates its response. These parameters fall into four categories:

1. **Sampling** — controls randomness and creativity
2. **Context** — controls memory and prompt handling
3. **Repetition** — controls output diversity
4. **Performance** — controls hardware utilisation

Understanding these parameters is essential for benchmarking, because the same model can behave very differently depending on how it is configured.

```json
{
  "model": "qwen3.6:27b",
  "prompt": "Your prompt here",
  "stream": false,
  "options": {
    "temperature": 0.7,
    "top_p": 0.9,
    "top_k": 40,
    "num_predict": 512,
    "num_ctx": 4096,
    "repeat_penalty": 1.1,
    "seed": -1
  }
}
```

---

## 1. Sampling Parameters

These control how the model picks its next token at each step. This is the core of LLM "creativity".

---

### `temperature`

| | |
|---|---|
| **Default** | `0.8` |
| **Range** | `0.0` – `2.0` |
| **Suggested for benchmarking** | `0.0` |
| **Suggested for creative tasks** | `0.7` – `1.0` |

**What it does:**

At each generation step, the model produces a probability distribution over all possible next tokens. `temperature` scales this distribution before sampling:

- **temperature = 0** — always pick the single highest-probability token (greedy decoding). Fully deterministic. Same input = same output every time.
- **temperature = 1** — use the raw probability distribution as-is.
- **temperature > 1** — flatten the distribution, making unlikely tokens more probable. More surprising, creative, and chaotic output.
- **temperature < 1** — sharpen the distribution, making the most likely tokens even more dominant. More focused and conservative output.

**Why it matters:**

For benchmarking speed (tok/s, TTFT), `temperature` has no effect — speed is purely a hardware metric.

For benchmarking **quality**, setting `temperature: 0` is critical because it makes outputs fully reproducible. If you run the same quality question twice at `temperature: 0.8`, you may get slightly different answers each time, making it impossible to do a fair comparison.

**Rule of thumb:**
- Benchmarking / evaluation → `0`
- Factual Q&A, coding → `0.1` – `0.3`
- General chat → `0.7`
- Creative writing, brainstorming → `0.8` – `1.2`

---

### `top_p` (Nucleus Sampling)

| | |
|---|---|
| **Default** | `0.9` |
| **Range** | `0.0` – `1.0` |
| **Suggested for benchmarking** | `1.0` (disabled) or leave default |
| **Suggested for general use** | `0.9` |

**What it does:**

Instead of considering all possible next tokens, `top_p` limits the candidate pool to the smallest set of tokens whose cumulative probability adds up to `p`.

Example with `top_p: 0.9`:
- Token A: 50% probability
- Token B: 25% probability
- Token C: 15% probability → cumulative = 90% ✅ stop here
- Token D: 5%, Token E: 5% → excluded

Only tokens A, B, C are candidates. The model then samples from these three based on their relative probabilities.

**Why it matters:**

`top_p` prevents the model from ever picking very low-probability (nonsensical) tokens, even at high temperatures. It is a safety net against incoherent output.

Setting `top_p: 1.0` disables nucleus sampling — all tokens are candidates.

**Interaction with temperature:** These two parameters are typically used together. A common pairing:
- `temperature: 0.7, top_p: 0.9` — good balance of quality and creativity
- `temperature: 0, top_p: 1.0` — fully deterministic greedy decoding

---

### `top_k`

| | |
|---|---|
| **Default** | `40` |
| **Range** | `1` – unlimited |
| **Suggested for benchmarking** | `1` (greedy) or leave default |
| **Suggested for general use** | `40` |

**What it does:**

Limits the candidate next tokens to only the top K highest-probability tokens, regardless of what those probabilities are.

Example with `top_k: 5`:
- Only the 5 highest-probability tokens are considered
- All others are excluded from sampling

**Difference from top_p:**

- `top_k` = fixed number of candidates
- `top_p` = variable number of candidates based on probability mass

`top_k: 1` is equivalent to greedy decoding (always pick the single best token), similar to `temperature: 0`.

**Why it matters:**

`top_k` is a hard ceiling. Even if there are 100 tokens with meaningful probability, only the top 40 are ever considered. This prevents very unlikely tokens from sneaking in.

For most use cases, the default of 40 is sensible. Lower values make output more focused; higher values allow more diversity.

---

### `seed`

| | |
|---|---|
| **Default** | `-1` (random) |
| **Range** | Any integer |
| **Suggested for benchmarking** | `42` (or any fixed value) |
| **Suggested for production** | `-1` |

**What it does:**

Sets the random seed for the sampling process. With the same seed and same parameters, the model will produce identical output for identical inputs.

**Why it matters for benchmarking:**

When running quality tests across multiple models, fixing the seed eliminates one source of variance. Combined with `temperature: 0`, this makes quality comparisons fully reproducible and scientifically valid.

```json
"options": {
  "temperature": 0,
  "seed": 42
}
```

> ⚠️ Note: In our current benchmark (Note 002/003), we did **not** fix the seed. This is a known limitation — results may vary slightly between runs. Future benchmarks should fix the seed for quality tests.

---

## 2. Repetition Parameters

---

### `repeat_penalty`

| | |
|---|---|
| **Default** | `1.1` |
| **Range** | `1.0` – `2.0` (values below 1.0 encourage repetition) |
| **Suggested** | `1.1` – `1.3` |

**What it does:**

Applies a penalty to tokens that have already appeared in the output. A value of `1.0` means no penalty (repetition is allowed). Higher values make it increasingly unlikely the model will repeat itself.

**Why it matters:**

Without repeat penalty, some models (especially at low temperatures) will get stuck in loops, repeating the same phrase over and over. This is particularly noticeable with smaller quantized models.

**Caution:** Setting this too high (above 1.5) can cause the model to avoid useful repeated words like "the", "is", "not", which degrades quality.

---

### `repeat_last_n`

| | |
|---|---|
| **Default** | `64` |
| **Range** | `0` – `num_ctx` |
| **Suggested** | `64` – `256` |

**What it does:**

Controls how far back the repeat penalty looks. Only tokens within the last `n` positions are penalised for repetition.

- `0` = no repetition penalty applied
- `-1` = look at the entire context window

---

## 3. Context Parameters

---

### `num_ctx`

| | |
|---|---|
| **Default** | `2048` |
| **Range** | Model-dependent (Qwen3.6 supports up to 128K+) |
| **Suggested for general use** | `4096` – `8192` |
| **Suggested for long tasks** | `32768` |

**What it does:**

Sets the size of the context window — how many tokens (input + output combined) the model can "see" at once. This is the KV cache size.

**Why it matters:**

This is one of the most impactful parameters for both quality and performance:

- **Too small** → model loses track of earlier parts of the conversation or document
- **Too large** → uses more VRAM, increases TTFT (more tokens to prefill), slower generation

**VRAM impact:** KV cache grows linearly with `num_ctx`. Doubling `num_ctx` roughly doubles KV cache memory usage.

For our benchmark, we used the Ollama default (2048). For real-world use of Qwen3.6, setting `num_ctx: 8192` or higher is recommended to take advantage of the model's capabilities.

---

### `num_batch`

| | |
|---|---|
| **Default** | `512` |
| **Range** | `1` – `num_ctx` |
| **Suggested** | `512` – `1024` |

**What it does:**

Controls how many tokens are processed in parallel during the **prefill** phase (processing your input prompt). This directly affects **TTFT**.

- Higher `num_batch` → processes more tokens at once → faster TTFT for long prompts
- Lower `num_batch` → slower prefill, lower peak GPU utilisation

**Note:** This parameter has no effect on generation speed (tok/s), only on TTFT.

---

## 4. Performance Parameters

---

### `num_gpu`

| | |
|---|---|
| **Default** | `auto` (all available layers) |
| **Range** | `0` – number of model layers |
| **Suggested** | Leave as `auto` unless testing CPU offload |

**What it does:**

Controls how many model layers are loaded onto the GPU. The remaining layers run on CPU.

- `num_gpu: 0` = full CPU inference (very slow)
- `num_gpu: auto` = all layers on GPU (maximum speed)
- Partial values = hybrid CPU+GPU (useful when VRAM is insufficient)

On the GB10 with 121.6 GiB VRAM, all three Qwen3.6 models fit entirely on GPU, so this parameter is not a concern for our setup.

---

### `num_thread`

| | |
|---|---|
| **Default** | auto (matches physical CPU cores) |
| **Suggested** | Leave as `auto` |

**What it does:**

Number of CPU threads used for inference. Only relevant when running on CPU or in hybrid mode. On a GPU-primary setup like ours, this has minimal impact.

---

### `f16_kv`

| | |
|---|---|
| **Default** | `true` |
| **Options** | `true` / `false` |

**What it does:**

When `true`, the KV cache is stored in FP16 (16-bit). When `false`, it uses FP32 (32-bit), doubling KV cache memory usage with minimal quality benefit.

Keep this as `true` unless you have a specific reason to use FP32 precision for the cache.

---

## 5. Recommended Parameter Sets

### For Benchmarking Speed (tok/s, TTFT)
```json
"options": {
  "num_predict": 512,
  "temperature": 0.8,
  "num_ctx": 2048
}
```
*Keep temperature and sampling at default to reflect real-world conditions.*

### For Benchmarking Quality (reproducible)
```json
"options": {
  "num_predict": 400,
  "temperature": 0,
  "seed": 42,
  "num_ctx": 4096
}
```
*Fixed seed + temperature 0 = fully reproducible results for fair comparison.*

### For Real-World Use (general chat)
```json
"options": {
  "num_predict": 2048,
  "temperature": 0.7,
  "top_p": 0.9,
  "top_k": 40,
  "repeat_penalty": 1.1,
  "num_ctx": 8192
}
```

### For Creative Writing
```json
"options": {
  "num_predict": 2048,
  "temperature": 1.0,
  "top_p": 0.95,
  "top_k": 50,
  "repeat_penalty": 1.15,
  "num_ctx": 8192
}
```

---

## 6. Known Limitations in Our Benchmark (Note 003)

| Issue | Impact | Fix for Next Benchmark |
|-------|--------|----------------------|
| `seed` not fixed | Quality results not fully reproducible | Add `seed: 42` |
| `temperature` not fixed (default 0.8) | Speed runs have slight randomness | Set `temperature: 0` for quality tests |
| `num_ctx` at default (2048) | May truncate long outputs | Set `num_ctx: 4096` |
| `think: true` by default on Qwen3 | `response` field is empty | Always set `think: false` unless testing reasoning |

> 💡 **Qwen3-specific note:** Qwen3 models default to thinking mode (`think: true`), which routes the final answer through a separate `thinking` field rather than `response`. When `think: true`, the model uses significantly more tokens for internal reasoning before producing the final answer — this inflates token count and generation time. Always explicitly set `think: false` for standard inference unless you specifically want chain-of-thought reasoning.

---

## Summary

| Parameter | Most Important For | TL;DR |
|-----------|-------------------|-------|
| `temperature` | Quality consistency | Set 0 for benchmarks, 0.7 for chat |
| `seed` | Reproducibility | Fix to 42 for fair quality tests |
| `num_predict` | Speed / cost control | Match to expected output length |
| `num_ctx` | Long document / chat | Set 4096+ for real use |
| `top_p` | Output coherence | Leave at 0.9 default |
| `top_k` | Token diversity | Leave at 40 default |
| `repeat_penalty` | Avoiding loops | Leave at 1.1 default |
| `num_batch` | TTFT optimisation | Increase for long prompts |

---

*Next: Note 005 will explore Qwen3's thinking mode in depth — when to use it, how to read the output, and how it affects quality vs speed.*
