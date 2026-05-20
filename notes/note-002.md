# Note 002 — Qwen3.6 27B Benchmark: Methodology & Test Plan

**Date:** 2026-05-20  
**Hardware:** Asus Ascent GX10 / NVIDIA GB10 Blackwell / 121.6 GiB VRAM  
**Framework:** Ollama  
**Models Under Test:** Qwen3.6 27B (Q4_K_M, Q8_0, BF16)

---

## Objective

To systematically benchmark three quantization levels of the Qwen3.6 27B model running locally on the NVIDIA GB10 Blackwell GPU, measuring:

1. **TTFT** — Time to First Token (ms)
2. **tok/s** — Token generation speed (tokens per second)
3. **Quality** — Reasoning accuracy and output quality

The goal is to understand the real-world trade-offs between model size, quantization, speed, and output quality — helping decide which format is best suited for different use cases.

---

## Models Under Test

| Model Tag | Format | Size on Disk | Quantization | Expected Quality |
|-----------|--------|-------------|--------------|-----------------|
| `qwen3.6:27b` | Q4_K_M | 17 GB | 4-bit (K-quantized) | Moderate loss |
| `qwen3.6:27b-q8_0` | Q8_0 | 30 GB | 8-bit | Minimal loss |
| `qwen3.6:27b-bf16` | BF16 | 56 GB | None (full precision) | Lossless |

All three models run entirely on GPU (GB10 has 121.6 GiB VRAM — sufficient to hold all three individually).

---

## Test Environment

- **Host:** Asus Ascent GX10 (`gx10`)
- **OS:** Ubuntu Linux (aarch64)
- **GPU:** NVIDIA GB10 Blackwell
- **VRAM:** 121.6 GiB
- **Ollama API:** `http://localhost`
- **Remote access:** SSH over local network
- **Benchmark script:** `~/benchmark.sh` (Bash + curl + Python3)

---

## Metrics Explained

### TTFT — Time to First Token
Time from sending the API request to receiving the first generated token. Measured in milliseconds.

- Includes: model prefill (processing the input prompt), scheduling latency
- Does NOT include: model load time (warmup handles this)
- Lower is better
- Typically scales with prompt length

### tok/s — Generation Speed
Number of tokens generated per second during the output phase.

- Calculated as: `eval_count / (eval_duration in seconds)`
- Measured from Ollama's native response metadata
- Higher is better
- The most important metric for interactive use

### Quality Score
Assessed by comparing model outputs against known correct answers on three reasoning tasks. No automated scoring — answers are reviewed manually post-benchmark.

---

## Prompt Design

### Speed Test Prompts

Each category contains **3 questions**. Each question is run **3 times** to average out variance from GPU scheduling and cache state.

#### 🟢 Short Prompts
Expected output: 1–2 sentences. Tests raw generation speed with minimal context.

| ID | Prompt |
|----|--------|
| S1 | What is the capital of France? |
| S2 | What is the difference between RAM and ROM? |
| S3 | In one sentence, what does HTTP stand for and what is it used for? |

#### 🟡 Medium Prompts
Expected output: 150–300 words. Tests sustained generation with moderate reasoning.

| ID | Prompt |
|----|--------|
| M1 | Explain the difference between TCP and UDP protocols in networking. Include when you would use each one. |
| M2 | What is the difference between symmetric and asymmetric encryption? Give a real-world example of each. |
| M3 | Explain what a REST API is and how it differs from GraphQL. When would you choose one over the other? |

#### 🔴 Long Prompts
Expected output: 400–1000 words. Tests sustained generation, instruction following, and structure.

| ID | Prompt |
|----|--------|
| L1 | System design: Real-time collaborative document editing (like Google Docs) — covering architecture, CRDT vs OT, backend stack, scalability, failure points |
| L2 | DevOps design: CI/CD pipeline for microservices e-commerce — covering branching model, build/test automation, Kubernetes, deployment strategy, monitoring |
| L3 | Creative writing: Write a 1000-word essay on the topic "World of 2046" — describe life, technology, society, and environment vividly |

---

### Quality Test Prompts

Run **1 time each**, across all 3 models. Assessed for reasoning correctness.

| ID | Prompt | Correct Answer |
|----|--------|----------------|
| Q1 | A train travels A→B at 80 km/h, returns at 120 km/h. What is the average speed? Show work. | **96 km/h** (harmonic mean, not 100) |
| Q2 | Three mislabeled boxes (apples / oranges / both). You pick 1 fruit from 1 box. How do you identify all labels? | Pick from the "Apples+Oranges" box — since all labels are wrong, that box is either apples-only or oranges-only. Its content reveals its true label; the rest follow by elimination. |
| Q3 | You have a 3-litre jug and a 5-litre jug. Measure exactly 4 litres. Show each step. | Fill 5L → pour into 3L → 2L left in 5L → empty 3L → pour 2L into 3L → fill 5L → pour 1L into 3L (now full) → 4L remains in 5L jug |

---

## Test Matrix

| Phase | Prompts | Runs per Prompt | Total Runs (per model) |
|-------|---------|-----------------|----------------------|
| Short speed | 3 | 3 | 9 |
| Medium speed | 3 | 3 | 9 |
| Long speed | 3 | 3 | 9 |
| Quality | 3 | 1 | 3 |
| **Per model total** | | | **30** |
| **3 models total** | | | **90** |

---

## Execution Protocol

1. **Warmup** — Each model runs a trivial prompt (`"hi"`) before measurement begins, to ensure the model is fully loaded into VRAM.
2. **Speed runs** — Short → Medium → Long prompts, 3 runs each, with a 2-second pause between runs.
3. **Quality run** — 3 quality prompts, 1 run each, with max_tokens set to 800.
4. **Model unload** — After each model completes, Ollama is instructed to unload the model before loading the next, preventing VRAM contamination between tests.
5. **Sleep** — 5-second pause between models to allow full cleanup.

---

## Output Format

Results are saved as JSON (`~/benchmark_results/qwen3_benchmark_TIMESTAMP.json`) with one record per run:

```json
{
  "model": "qwen3.6:27b",
  "prompt_label": "short_S1",
  "run": 1,
  "ttft_ms": 142.3,
  "toks_per_sec": 87.4,
  "eval_count": 12,
  "total_ms": 278,
  "response_preview": "The capital of France is Paris."
}
```

Quality results include the full response text for manual review.

---

## What We Expect to Find

| Metric | Q4_K_M | Q8_0 | BF16 |
|--------|--------|------|------|
| tok/s | Fastest | Middle | Slowest |
| TTFT | Shortest | Middle | Longest |
| Quality | Some degradation | Near-lossless | Reference quality |
| VRAM used | ~17 GB | ~30 GB | ~56 GB |

The hypothesis: **Q8_0 hits the sweet spot** — close to BF16 quality, meaningfully faster than BF16, and fits comfortably in VRAM with headroom for KV cache.

---

## Next Note

Note 003 will contain the actual benchmark results, charts, and conclusions.

---

*Benchmarked on Asus Ascent GX10 — one of the first consumer-grade GB10 Blackwell systems available.*
