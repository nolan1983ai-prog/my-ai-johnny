# Note 008 — Qwopus3.6-27B-v2-MTP: Why It's Better and Jackrong's Full Benchmark

**Date:** 2026-05-25
**Hardware:** Asus Ascent GX10 / NVIDIA GB10 Blackwell / 121.6 GiB VRAM
**Topic:** What makes Qwopus3.6-27B-v2-MTP better than vanilla Qwen3.6-27B, and a full breakdown of Jackrong's author-reported benchmark

> ⚠️ **Data Source Notice:** All benchmark numbers in this note are **author-reported** by Jackrong (the model creator), run on a GB10 server. These figures have not been independently verified by this journal. Reference: https://huggingface.co/Jackrong/Qwopus3.6-27B-v2-MTP-GGUF

---

## 1. What is Qwopus3.6-27B-v2-MTP-Q4_K_M?

It is a community fine-tuned model built on top of Alibaba's **Qwen3.6-27B** base, created by developer **Jackrong**. The full name breaks down as:

| Part | Meaning |
|------|---------|
| `Qwopus` | Project name (Qwen + Opus distillation lineage) |
| `3.6` | Based on Qwen 3.6 generation |
| `27B` | 27 billion parameters |
| `v2` | Second version of this fine-tune |
| `MTP` | Multi-Token Prediction — speculative decoding for speed |
| `Q4_K_M` | GGUF quantization: 4-bit, K-quantized, medium variant |

---

## 2. Why is it Better Than Original Qwen3.6-27B?

Two independent improvements are stacked on top of the base model:

### 2.1 Trace Inversion — Better Reasoning Quality

**The problem with standard distillation:**
When smaller models are trained using Claude/GPT outputs directly, they only learn the *conclusions* — the intermediate reasoning steps are compressed or missing. The result: a model that gives right answers but doesn't actually "think" step-by-step.

**Jackrong's solution — Trace Inversion:**

```
Step A: Train a small Trace-Inverter-4B model
  → Feed it Claude-4.7-Max compressed outputs ("Reasoning Bubbles")
  → It learns to reconstruct the full step-by-step reasoning chain

Step B: Apply Trace-Inverter-4B to Claude-4.7-Max outputs
  → Compressed bubble → Full learnable CoT (Chain of Thought)

Step C: Fine-tune Qwen3.6-27B on these reconstructed traces
  → Model learns deep reasoning, not just surface answers
```

**Result:** The model internalises *how* Claude reasons, not just *what* Claude answers. Shorter output tokens with equivalent or better correctness.

### 2.2 Three-Stage Curriculum Learning

Fine-tuning was done progressively to avoid format collapse:

| Stage | Context Length | Focus |
|-------|---------------|-------|
| Stage 1: Format Inception | < 4,096 tokens | Stable `<think>` tag formatting |
| Stage 2: Complexity Expansion | 4,096–8,192 tokens | Hard logic + complex reasoning chains |
| Stage 3: Long-Context SFT | Up to 32K tokens | Multi-turn, ultra-long reasoning (10% short replay) |

### 2.3 MTP (Multi-Token Prediction) — Speed

MTP adds a speculative decoding head that predicts multiple tokens in parallel. This is layered on top of the Qwopus v2 base weights.

**Effect on GB10 (author-reported):**
- Qwen3.6-27B base: **6.29 tok/s**
- Qwopus MTP: **10.46 tok/s** (+66% speed)
- 30-question run: 14,901s → 6,488s (**2.3× faster**)
- Token output: 93,802 → 67,862 (**28% more concise**)

---

## 3. Jackrong's Benchmark — Full Details

### 3.1 Test Environment

| Setting | Value |
|---------|-------|
| Hardware | GB10 dedicated server (same as GX10) |
| Framework | llama-server (GGUF stack) |
| Context window | 49,152 tokens |
| Temperature | 1.0 |
| Top-p | 0.95 |
| Max tokens | No cap (request-bounded) |
| Request format | `/v1/chat/completions` |
| Models compared | Qwen3.6-27B (base) vs Qwopus3.6-27B-v2-MTP |
| Questions | 30 total across 5 domains |

### 3.2 Domain-Level Summary (Author-Reported)

| Domain | Questions | Qwen3.6 T/s | MTP T/s | Speed Gain | Token Reduction |
|--------|-----------|-------------|---------|-----------|----------------|
| Logic | 5 | 6.33 | 10.77 | **2.31×** | -26.3% |
| Coding | 7 | 6.26 | 10.27 | **2.25×** | -27.3% |
| DevOps | 6 | 6.29 | 10.39 | **2.31×** | -28.5% |
| Math | 8 | 6.29 | 11.00 | **2.35×** | -25.6% |
| Edge | 4 | 6.48 | 8.28 | **2.27×** | -43.6% |

### 3.3 Full 30-Question Breakdown (Author-Reported)

| Q | Domain | Task | Qwen T/s | Qwen Time | Qwen Tokens | MTP T/s | MTP Time | MTP Tokens | Result |
|---|--------|------|----------|-----------|-------------|---------|----------|------------|--------|
| Q1 | Logic | Wrong-label coin boxes | 6.36 | 9.4 min | 3,569 | 11.40 | 2.3 min | 1,530 | **4.16× faster**; much more concise |
| Q2 | Logic | Engineer deployment ordering | 6.39 | 6.1 min | 2,349 | 10.98 | 3.1 min | 2,034 | **1.98× faster**; more concise |
| Q3 | Logic | Self-referential truth card | 6.37 | 7.8 min | 2,990 | 10.83 | 4.5 min | 2,942 | **1.72× faster**; similar length |
| Q4 | Logic | Three switches and bulbs | 6.32 | 3.6 min | 1,342 | 10.44 | 1.6 min | 999 | **2.21× faster**; more concise |
| Q5 | Logic | HH vs TH stopping probability | 6.30 | 11.6 min | 4,367 | 10.62 | 5.2 min | 3,266 | **2.25× faster**; more concise |
| Q6 | Coding | Streaming top-k frequency | 6.28 | 13.8 min | 5,210 | 9.95 | 13.3 min | 7,917 | **1.04× faster**; more expansive |
| Q7 | Coding | Thread-safe TTL cache | 6.28 | 18.6 min | 7,009 | 10.64 | 5.3 min | 3,367 | **3.52× faster**; much more concise |
| Q8 | Coding | Interval merge implementation | 6.25 | 11.2 min | 4,203 | 10.83 | 3.3 min | 2,157 | **3.36× faster**; much more concise |
| Q9 | Coding | Streaming CSV to JSONL | 6.26 | 16.5 min | 6,200 | 10.62 | 5.9 min | 3,741 | **2.81× faster**; more concise |
| Q10 | Coding | C++17 LRU cache | 6.27 | 13.1 min | 4,920 | 10.15 | 6.0 min | 3,644 | **2.18× faster**; more concise |
| Q11 | Coding | Highest-paid employee SQL | 6.29 | 6.1 min | 2,283 | 10.37 | 2.4 min | 1,475 | **2.54× faster**; more concise |
| Q12 | Coding | Atomic Bash backup | 6.28 | 12.1 min | 4,545 | 10.33 | 4.4 min | 2,695 | **2.76× faster**; much more concise |
| Q13 | DevOps | Nginx reverse proxy | 6.29 | 10.4 min | 3,924 | 10.88 | 2.8 min | 1,821 | **3.70× faster**; much more concise |
| Q14 | DevOps | Linux service OOM diagnosis | 6.29 | 9.9 min | 3,727 | 9.96 | 4.9 min | 2,888 | **2.04× faster**; more concise |
| Q15 | DevOps | systemd worker unit | 6.29 | 8.0 min | 3,023 | 10.39 | 3.3 min | 2,037 | **2.43× faster**; more concise |
| Q16 | DevOps | Kubernetes rollback runbook | 6.32 | 6.3 min | 2,387 | 10.36 | 2.9 min | 1,820 | **2.14× faster**; more concise |
| Q17 | DevOps | Docker CMD vs ENTRYPOINT | 6.33 | 5.4 min | 2,028 | 10.78 | 2.9 min | 1,892 | **1.82× faster**; more concise |
| Q18 | DevOps | Prometheus pull monitoring | 6.32 | 7.4 min | 2,818 | 10.67 | 3.7 min | 2,342 | **2.02× faster**; more concise |
| Q19 | Math | Derivative and critical point | 6.32 | 8.7 min | 3,274 | 12.06 | 3.7 min | 2,631 | **2.37× faster**; more concise |
| Q20 | Math | Linear system solve | 6.32 | 10.7 min | 4,065 | 11.91 | 4.2 min | 2,976 | **2.57× faster**; more concise |
| Q21 | Math | Different-colour probability | 6.28 | 3.9 min | 1,472 | 10.18 | 49.6 s | 490 | **4.74× faster**; much more concise |
| Q22 | Math | 2×2 eigen decomposition | 6.31 | 12.3 min | 4,662 | 11.28 | 4.5 min | 3,058 | **2.72× faster**; more concise |
| Q23 | Math | Induction proof | 6.32 | 5.8 min | 2,211 | 11.53 | 1.7 min | 1,193 | **3.34× faster**; much more concise |
| Q24 | Math | Bayes disease test | 6.34 | 5.0 min | 1,878 | 11.38 | 3.2 min | 2,156 | **1.56× faster**; more expansive |
| Q25 | Math | Integration by parts | 6.29 | 5.5 min | 2,064 | 11.80 | 3.5 min | 2,493 | **1.55× faster**; more expansive |
| Q26 | Math | Central Limit Theorem | 6.24 | 8.8 min | 3,289 | 8.26 | 4.1 min | 2,046 | **2.12× faster**; more concise |
| Q27 | Edge | Strict JSON output | 6.32 | 3.6 min | 1,350 | 10.43 | 23.1 s | 225 | **9.28× faster**; much more concise |
| Q28 | Edge | Exact token pattern | 6.37 | 52.4 s | 328 | 12.15 | 29.9 s | 345 | **1.75× faster**; similar length |
| Q29 | Edge | Forbidden-word explanation | 6.71 | 5.1 min | 2,040 | 7.62 | 3.5 min | 1,573 | **1.47× faster**; more concise |
| Q30 | Edge | Ignore noisy input | 6.35 | 44.5 s | 275 | 10.94 | 11.4 s | 109 | **3.89× faster**; much more concise |

### 3.4 Notable Observations

**Biggest speed wins (MTP effect strongest):**
- Q27 Edge (Strict JSON output): **9.28× faster** — constrained output is where MTP shines most
- Q21 Math (probability): **4.74× faster**
- Q1 Logic (coin boxes): **4.16× faster**

**MTP occasionally produces more tokens (expansive):**
- Q6 Coding (streaming top-k): MTP produced 7,917 tokens vs 5,210 — MTP was more thorough here
- Q24/Q25 Math: slightly more expansive answers

**Edge cases benefit most from MTP:**
- Strict/constrained output tasks (JSON, token patterns) see the highest speed gains — MTP predicts repetitive/structured tokens very efficiently

---

## 4. Summary — Why Use Qwopus MTP Over Vanilla Qwen3.6-27B?

| Dimension | Qwen3.6-27B | Qwopus3.6-27B-v2-MTP |
|-----------|-------------|----------------------|
| Reasoning quality | Good | Better (Trace Inversion) |
| Output conciseness | Verbose | ~28% fewer tokens |
| Speed | 6.29 tok/s | 10.46 tok/s (+66%) |
| Time per 30 tasks | ~4.1 hours | ~1.8 hours |
| Structured output | Standard | Significantly faster |
| Availability | Ollama registry | GGUF only (llama.cpp required) |

**Bottom line:** Qwopus MTP is a community model that stacks reasoning quality improvement (Trace Inversion) with a speed layer (MTP). For users running on GB10-class hardware with llama.cpp, it is strictly better than the base model on both quality and speed dimensions — with no additional VRAM cost.

---

*Next step: run our own benchmark on GX10 to independently verify these claims.*
