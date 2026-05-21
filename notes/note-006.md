# Note 006 — Revised Benchmark Results: Qwen3.6 27B Q4_K_M vs Q8_0 vs BF16

**Date:** 2026-05-21  
**Hardware:** Asus Ascent GX10 / NVIDIA GB10 Blackwell / 121.6 GiB VRAM  
**Framework:** Ollama  
**Supersedes:** Note 003 (methodology revised per Note 005)  
**Models tested:** qwen3.6:27b (Q4_K_M), qwen3.6:27b-q8_0, qwen3.6:27b-bf16

---

## Executive Summary

The revised benchmark confirms clear speed stratification between quantisation levels, with higher precision meaning meaningfully slower throughput. Q4_K_M runs at roughly 2.5× the token/s of BF16, and Q8_0 sits in the middle at ~1.7×. TTFT scales more gently with quantisation because it is dominated by prompt processing, not weight-loading speed.

**Quality data is incomplete** — the benchmark script captured only partial responses (200-token limit) for question A1 in Q8_0 and BF16, while all Q4_K_M quality files are empty (capture bug during first model run). The A1 responses that were captured look correct and consistent across 3 runs, suggesting identical output with seed=42/temperature=0.

**VRAM measurement failed** — nvidia-smi returned `[N/A]` for `memory.used` on all three models. This is a known limitation with the GB10 Blackwell's unified/shared memory architecture; traditional VRAM counters do not apply. GPU utilisation tracked at 95–96% during inference, confirming the hardware was working.

Confidence: **medium** on speed results (methodology sound, 8 runs), **low** on quality (incomplete capture), **N/A** on VRAM.

---

## Speed Results

### Methodology (what changed from Note 003)
- Fixed: `seed=42`, `temperature=0`, `num_predict=200`, `num_ctx=4096`
- 1 warm-up run discarded per model per prompt type
- 8 measured runs per prompt type (short/medium/long)
- VRAM attempted via `nvidia-smi` mid-inference (see VRAM section below)

### Results Table

| Model | Format | Prompt Type | Mean tok/s | ± stdev | Min | Max | TTFT mean (ms) | ± stdev | TTFT min | TTFT max |
|-------|--------|-------------|-----------|---------|-----|-----|----------------|---------|----------|----------|
| qwen3.6:27b | Q4_K_M | short | 11.11 | 0.10 | 10.86 | 11.17 | 152.4 | 8.4 | 144.2 | 168.1 |
| qwen3.6:27b | Q4_K_M | medium | 11.12 | 0.00 | 11.11 | 11.12 | 196.1 | 3.3 | 194.2 | 203.8 |
| qwen3.6:27b | Q4_K_M | long | 11.11 | 0.01 | 11.11 | 11.12 | 528.6 | 9.4 | 515.8 | 542.0 |
| qwen3.6:27b-q8_0 | Q8_0 | short | 7.39 | 0.01 | 7.37 | 7.39 | 202.6 | 3.3 | 198.7 | 208.6 |
| qwen3.6:27b-q8_0 | Q8_0 | medium | 7.44 | 0.00 | 7.44 | 7.45 | 254.6 | 7.3 | 247.2 | 268.4 |
| qwen3.6:27b-q8_0 | Q8_0 | long | 7.45 | 0.00 | 7.44 | 7.45 | 721.1 | 5.6 | 716.0 | 731.0 |
| qwen3.6:27b-bf16 | BF16 | short | 4.43 | 0.01 | 4.41 | 4.45 | 292.1 | 4.2 | 287.2 | 299.4 |
| qwen3.6:27b-bf16 | BF16 | medium | 4.44 | 0.00 | 4.44 | 4.45 | 341.9 | 1.9 | 339.6 | 345.8 |
| qwen3.6:27b-bf16 | BF16 | long | 4.44 | 0.01 | 4.44 | 4.45 | 559.2 | 6.1 | 551.1 | 567.9 |

### Speed Ratio Analysis

Using medium-prompt means as the reference (most stable stdev):

| Comparison | Tok/s ratio | 95% CI (propagated stdev) |
|------------|------------|--------------------------|
| Q4_K_M vs BF16 | 11.12 / 4.44 = **2.505×** | ±0.001 (essentially exact — stdev ~0) |
| Q8_0 vs BF16 | 7.44 / 4.44 = **1.676×** | ±0.001 |
| Q4_K_M vs Q8_0 | 11.12 / 7.44 = **1.495×** | ±0.001 |

TTFT ratios (medium prompt):
- Q4_K_M TTFT is 196ms vs BF16 341ms → Q4_K_M is **1.74× faster** TTFT
- Q8_0 TTFT 255ms vs BF16 341ms → Q8_0 is **1.34× faster** TTFT

Interesting observation: Q4_K_M short-prompt TTFT (152ms) is noticeably lower stdev-stable, but the long-prompt TTFT (529ms) is significantly faster than Q8_0 long (721ms) — suggesting prompt-processing cost also scales with quantisation precision.

**Notable:** Token/s are remarkably stable across prompt lengths within each model (11.11–11.12 for Q4_K_M), which is expected with fixed `num_predict=200` but confirms the methodology is sound.

---

## VRAM Results (Measured)

**nvidia-smi returned `[N/A]` for `memory.used` across all three models throughout the entire benchmark run.**

GPU utilisation confirmed active at 95–96% during inference, 0% at rest — so the hardware *was* being used, but the GB10 Blackwell's unified memory architecture does not expose traditional discrete VRAM counters via nvidia-smi. This is a known limitation with the GB10 / GH200-class chips where CPU and GPU share a unified memory pool.

| Model | Peak memory.used | GPU utilisation (peak) |
|-------|-----------------|----------------------|
| qwen3.6:27b (Q4_K_M) | N/A (unified memory) | 95–96% |
| qwen3.6:27b-q8_0 | N/A (unified memory) | 95–96% |
| qwen3.6:27b-bf16 | N/A (unified memory) | 95–96% |

**For future benchmarks:** Use `tegrastats` or `/proc/meminfo` combined with Ollama's process RSS to estimate actual memory consumption on this platform. Alternatively, query Ollama's `/api/ps` endpoint which may report model size in RAM.

---

## Quality Results

### Capture Issues

The benchmark script had two problems:
1. **Q4_K_M (first model run):** All 7 quality files are empty — the script appears to have failed to capture responses for the first model processed. Likely a race condition or output redirect bug during the initial run.
2. **All models, questions A2–C2:** Only run headers (`--- Run 1 ---`) captured, no actual response text. Possibly the 200-token `num_predict` limit caused early stopping before the script's output parsing completed, or the API returned empty body for these shorter-answer questions.

Only **A1 (parking spaces)** has actual response content, and only for Q8_0 and BF16.

### Q8_0 — Question A1 (captured, truncated at 200 tokens)

Response prefix (all 3 runs identical with seed=42/temp=0):
> "Here's the step-by-step working: 1. **Find the total number of spaces:** 48 (Floor 1) + 35 (Floor 2) + 27 (Floor..."

The response starts correctly (summing floors toward 110 total), but is truncated before reaching the answer. Based on the method being shown, the model is on track for **44** (correct answer). **Consistency: perfect** (3/3 identical due to deterministic settings).

### BF16 — Question A1 (captured, truncated at 200 tokens)

Response prefix (all 3 runs identical):
> "Here's the step-by-step working: 1. **Find the total number of spaces:** 48 (Floor 1) + 35 (Floor 2) +..."

Slightly less text captured than Q8_0 (slightly different tokenisation boundary). Same approach, same trajectory toward correct answer. **Consistency: perfect** (3/3 identical).

### Q4_K_M — All questions

All files empty due to capture bug. **No data.**

### Scoring Summary

| Model | A1 | A2 | A3 | B1 | B2 | C1 | C2 | Total |
|-------|----|----|----|----|----|----|-----|-------|
| Q4_K_M | N/A | N/A | N/A | N/A | N/A | N/A | N/A | N/A |
| Q8_0 | ~1.0* | N/A | N/A | N/A | N/A | N/A | N/A | ~1.0/7 |
| BF16 | ~1.0* | N/A | N/A | N/A | N/A | N/A | N/A | ~1.0/7 |

*Score inferred from response prefix showing correct methodology; full answer not confirmed.

Quality benchmark is **effectively inconclusive** for this run.

---

## Comparison vs Note 003

Note 003 used variable temperature and inconsistent seeding, making run-to-run comparisons unreliable. With the revised methodology:

- **Speed numbers are now reproducible:** Near-zero stdev on medium/long prompts confirms deterministic token generation with fixed seed+temperature. Note 003 likely had higher variance.
- **TTFT short-prompt variance** is higher (stdev 4–8ms) as expected, since short prompts trigger less predictable prefill paths.
- **No valid quality comparison possible** due to capture failures.
- **VRAM comparison not possible** due to `[N/A]` on GB10.

The key new finding from the revised methodology: **the speed ratio Q4_K_M:BF16 is a clean 2.5×** with extremely high precision (±0.001). This is consistent with the theoretical ~2× weight size difference plus memory bandwidth overhead from BF16.

---

## Known Remaining Limitations

1. **Quality benchmark failure** — Only 1/7 questions partially captured for 2/3 models. No valid quality comparison possible. Needs rerun with increased `num_predict` (≥500) and fixed output capture logic.
2. **VRAM measurement not working** on GB10 — nvidia-smi unified memory counters return N/A. Need alternative measurement strategy for this platform.
3. **200-token generation limit** — Artificial cap may cut off reasoning chains, particularly for math problems (A1–A3) where models show full working. Quality scores are based on partial evidence.
4. **Single hardware configuration** — All results are GB10-specific. Not generalisable to other GPU types without rerunning.
5. **Ollama version and model loading not documented** — Should be captured in future runs.
6. **Context window (4096) is minimal** — Real workloads with longer contexts may show different TTFT scaling.

---

## Conclusions & Updated Recommendations

On this hardware and test configuration:

**Speed:** Q4_K_M is the clear winner at ~11.1 tok/s vs ~4.4 tok/s for BF16 — a stable, reproducible 2.5× advantage. For interactive use cases on the GX10, Q4_K_M is strongly preferred unless quality differences are demonstrated.

**Quality:** Inconclusive from this run. Prior heuristic expectation is that BF16 ≥ Q8_0 ≥ Q4_K_M on complex reasoning tasks, but the margin may be small for well-optimised quantisation like Q4_K_M. A clean quality rerun is needed.

**Recommendation (with appropriate hedging):** For this specific test setup, use Q4_K_M for throughput-sensitive tasks. If quality is paramount and 2.5× latency penalty is acceptable, BF16 may be preferable — but this is speculative until quality data is obtained.

*← [Back to Index](../README.md)*
