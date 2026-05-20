# Note 003 — Qwen3.6 27B Benchmark Results — Q4_K_M vs Q8_0 vs BF16

**Date:** 2026-05-21
**Hardware:** Asus Ascent GX10 / NVIDIA GB10 Blackwell / 121.6 GiB VRAM
**Framework:** Ollama
**Models tested:** `qwen3.6:27b` (Q4_K_M), `qwen3.6:27b-q8_0` (Q8_0), `qwen3.6:27b-bf16` (BF16)

---

## Executive Summary

This note presents the full benchmark results for three quantization formats of Qwen3.6 27B running on the Asus Ascent GX10 (NVIDIA GB10 Blackwell, 121.6 GiB VRAM). Two dimensions were measured:

1. **Speed** — tokens per second and time-to-first-token (TTFT) across short, medium, and long prompts
2. **Quality** — reasoning ability assessed via three logic and mathematics questions (run with `think: false`)

**Key findings:** Q4_K_M is the fastest at ~11.15 tok/s; Q8_0 offers a balanced middle ground at ~7.26 tok/s; BF16 is the slowest at ~4.42 tok/s. On quality, all three formats performed identically — correctly solving all three reasoning questions. This suggests quantization from BF16 → Q8_0 → Q4_K_M introduces no measurable quality degradation on standard reasoning tasks.

---

## Speed Results

### Benchmark Methodology

- 3 prompt types: **short**, **medium**, **long**
- 3 runs per prompt type (averaged)
- 4-second pause between runs; 10-second cooldown between prompt types
- Metrics recorded: tokens per second (tok/s) and time-to-first-token (TTFT in ms)
- `seed` and `temperature` were **not fixed** (see Known Limitations)

### Results Table

| Model | Format | Short tok/s | Short TTFT | Medium tok/s | Medium TTFT | Long tok/s | Long TTFT | **Avg tok/s** | **Avg TTFT** |
|-------|--------|-------------|------------|--------------|-------------|------------|-----------|---------------|--------------|
| `qwen3.6:27b` | Q4_K_M | 11.23 | 148.5ms | 11.12 | 149.7ms | 11.11 | 206.8ms | **11.15** | **168ms** |
| `qwen3.6:27b-q8_0` | Q8_0 | 7.26 | 212.0ms | 7.23 | 228.2ms | 7.29 | 263.5ms | **7.26** | **235ms** |
| `qwen3.6:27b-bf16` | BF16 | 4.43 | 289.7ms | 4.42 | 295.0ms | 4.42 | 346.5ms | **4.42** | **310ms** |

### Key Observations from Speed Data

1. **Q4_K_M is 2.52× faster than Q8_0** (11.15 vs 7.26 tok/s) — a significant throughput advantage for interactive use.

2. **Q8_0 is 1.64× faster than BF16** (7.26 vs 4.42 tok/s) — using INT8 quantization costs only minimal quality while nearly doubling speed vs. full precision.

3. **Q4_K_M is 2.52× faster than BF16** overall — impressive performance for a compressed format.

4. **TTFT scales with model size** — heavier quantization formats have higher first-token latency. BF16's 310ms TTFT vs Q4_K_M's 168ms means noticeably longer "thinking pauses" before output begins.

5. **Prompt length has minimal impact on tok/s** — all three models sustain nearly constant throughput across short/medium/long prompts, suggesting the GB10 Blackwell GPU handles different context lengths efficiently.

6. **Long prompts increase TTFT** across all models (expected — prefill phase is longer), but the increase is moderate (~40–58ms from short to long).

---

## Quality Results

### Benchmark Methodology

- Quality test run with **`think: false`** (see Important Note below)
- 3 questions covering reasoning, logic, and problem-solving
- Answers captured from the `response` field of each completion
- Each model answered each question once (no seed/temperature control — see Known Limitations)

> ⚠️ **Important Note on `think: false`:**
> Qwen3 models default to thinking mode, where the final reasoned answer is placed in the `thinking` field and `response` is left empty. The quality benchmark was run with `think: false` to ensure answers appeared in the `response` field. Running with `think: true` requires reading from a different output field and may produce different (potentially better) quality results. This will be covered in **Note 005**.

---

### Q1 — Train Average Speed (Harmonic Mean)

**Question:** A train travels from City A to City B at 80 km/h, and returns at 120 km/h. What is the average speed for the entire round trip?

**Correct answer:** 96 km/h (harmonic mean, not arithmetic mean)

| Model | Answer | Correct? |
|-------|--------|----------|
| `qwen3.6:27b` (Q4_K_M) | 96 km/h — used harmonic mean formula correctly; explicitly noted arithmetic mean is wrong | ✅ |
| `qwen3.6:27b-q8_0` | 96 km/h — full step-by-step derivation with LCM; correct formula applied | ✅ |
| `qwen3.6:27b-bf16` | 96 km/h — identified common pitfall (arithmetic mean); derived via total distance / total time | ✅ |

All three models correctly applied the harmonic mean and showed full working. Q4_K_M and Q8_0 both noted that `(80 + 120) / 2 = 100 km/h` is incorrect and explained why.

---

### Q2 — Mislabeled Boxes Logic Puzzle

**Question:** Three boxes are labeled "Apples", "Oranges", and "Mixed". All labels are wrong. By picking one fruit from one box, can you correctly label all three?

**Correct answer:** Pick from the box labeled "Mixed" (or "Apples and Oranges") — since it's mislabeled, it must be pure (apples only or oranges only). The result lets you deduce the other two by elimination.

| Model | Answer | Correct? |
|-------|--------|----------|
| `qwen3.6:27b` (Q4_K_M) | Pick from box labeled "Mixed" → deduce all three by elimination | ✅ |
| `qwen3.6:27b-q8_0` | Pick from "Apples and Oranges" box → full case analysis (A/B scenarios) | ✅ |
| `qwen3.6:27b-bf16` | Pick from "Both Apples and Oranges" box → step-by-step deduction with both cases | ✅ |

All three models arrived at the correct strategy and reasoning. Q8_0 and BF16 provided particularly thorough two-case (apple / orange pick) analyses.

---

### Q3 — 3L/5L Jug Water Measuring Puzzle

**Question:** Using a 3-litre jug and a 5-litre jug (unlimited water supply), measure exactly 4 litres.

**Correct answer:** Fill 5L → pour to 3L (5L has 2L left) → empty 3L → pour 2L into 3L → fill 5L again → pour into 3L until full (takes 1L) → 5L jug now has exactly 4L.

| Model | Answer | Correct? |
|-------|--------|----------|
| `qwen3.6:27b` (Q4_K_M) | Correct 6-step solution; 5L jug ends with 4L | ✅ |
| `qwen3.6:27b-q8_0` | Correct 6-step primary method + alternative "fill 3L first" method | ✅ |
| `qwen3.6:27b-bf16` | Correct 6-step primary method + alternative method; clear state tracking | ✅ |

All three models solved the puzzle correctly. Q8_0 and BF16 additionally provided the alternative 7-step solution (starting by filling the 3L jug first).

---

## Quality Assessment Summary

| Question | Q4_K_M | Q8_0 | BF16 |
|----------|--------|------|------|
| Q1: Train average speed (96 km/h) | ✅ Correct | ✅ Correct | ✅ Correct |
| Q2: Mislabeled boxes logic | ✅ Correct | ✅ Correct | ✅ Correct |
| Q3: 3L/5L jug puzzle | ✅ Correct | ✅ Correct | ✅ Correct |
| **Total Score** | **3/3** | **3/3** | **3/3** |

**Conclusion:** On these three reasoning tasks, all quantization levels performed identically. There is **no observable quality degradation** from BF16 → Q8_0 → Q4_K_M for standard logic and math reasoning questions.

---

## Known Limitations

1. **`think: false` mode only** — Qwen3 has a thinking mode (`think: true`) where extended chain-of-thought reasoning is placed in a `thinking` field. This benchmark only tested non-thinking mode. Thinking mode may produce significantly different (likely better) quality results and will be investigated in Note 005.

2. **Seed and temperature not fixed** — Runs were performed without fixing `seed` or `temperature`. Results may vary between runs due to sampling randomness. A rigorous quality benchmark should fix both parameters for reproducibility.

3. **Single-run quality evaluation** — Each model answered each question once. Statistical significance requires multiple runs per question.

4. **Limited question set** — 3 questions is a minimal sample. Results may not generalize to all reasoning categories (e.g., creative writing, coding, multi-step arithmetic, knowledge recall).

5. **Speed benchmark uses 3 prompts × 3 runs** — Averages are based on 9 data points per model; variance within each run type was not reported.

---

## Conclusions & Recommendations

### Speed vs Quality Trade-off

| Format | Speed | Quality (observed) | VRAM | Recommendation |
|--------|-------|--------------------|------|----------------|
| Q4_K_M | ⚡⚡⚡ 11.15 tok/s | ✅ Full score | ~17 GB | **Best for interactive/daily use** |
| Q8_0 | ⚡⚡ 7.26 tok/s | ✅ Full score | ~30 GB | **Best for quality-sensitive tasks** |
| BF16 | ⚡ 4.42 tok/s | ✅ Full score | ~56 GB | **Reference baseline only** |

### Practical Recommendation

For the Asus Ascent GX10 with 121.6 GiB VRAM:

- **Primary use:** `qwen3.6:27b` (Q4_K_M) — 2.52× faster than BF16, no quality loss on reasoning tasks, leaves massive VRAM headroom for other models or larger context windows.
- **High-stakes tasks:** `qwen3.6:27b-q8_0` — minimal speed penalty over Q4_K_M for potentially better nuanced generation quality.
- **BF16:** Only warranted if absolute maximum precision is required or for research comparisons.

> **Bottom line:** Q4_K_M wins on this hardware. On the GB10 Blackwell with ample VRAM, there is no resource pressure forcing BF16 — and Q4_K_M's 2.5× speed advantage makes real-time conversations significantly more fluid.

---

## Preview: Note 005 — Qwen3 Thinking Mode Deep Dive

Note 005 will explore **Qwen3's thinking mode** (`think: true`), which is the model's default behavior:

- Why `response` is empty when `think: true` (answer goes into `thinking` field)
- How to properly read thinking mode output in Ollama API
- Quality comparison: thinking mode vs non-thinking mode on the same Q1/Q2/Q3 questions
- Latency impact of extended chain-of-thought reasoning
- When to use thinking vs non-thinking mode for different task types

---

*← [Back to Index](../README.md)*
