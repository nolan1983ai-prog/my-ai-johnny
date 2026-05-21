# Note 005 — Revised Benchmark Test Plan: Qwen3.6 27B Quantization Comparison

**Date:** 2026-05-21
**Hardware:** Asus Ascent GX10 / NVIDIA GB10 Blackwell / 121.6 GiB VRAM
**Framework:** Ollama
**Purpose:** This note defines the revised, methodologically rigorous benchmark plan addressing all weaknesses identified in the expert review of Note 003. Results will be published in Note 006.

---

## Background: Why a Revised Plan?

Note 003 established a useful baseline but had several methodological weaknesses:

| Weakness | Impact |
|----------|--------|
| Seed and temperature not fixed | Quality results unreproducible |
| Single run per quality question | Statistically anecdotal |
| No standard deviation on speed | "2.52× faster" claim has no error bar |
| Quality sample size: 3 questions, 1 run each | Cannot support "no degradation" conclusion |
| Prompt token counts not defined | Not reproducible |
| VRAM from file size estimate, not measured | Inaccurate runtime figures |
| No warm-up run | First run may be slower (cold cache) |
| Classic/training-contaminated puzzles | Tests recall, not reasoning |

This revised plan fixes all of the above.

---

## Models Under Test

| Tag | Format | Expected VRAM |
|-----|--------|---------------|
| `qwen3.6:27b` | Q4_K_M | ~17 GB |
| `qwen3.6:27b-q8_0` | Q8_0 | ~30 GB |
| `qwen3.6:27b-bf16` | BF16 | ~56 GB |

---

## Test Dimensions

### 1. Speed Benchmark

**Goal:** Measure generation throughput and TTFT with proper statistical rigor.

**Parameters (fixed):**
- `temperature: 0` (deterministic)
- `seed: 42`
- `num_predict: 200` (output token count fixed)
- `num_ctx: 4096`

**Prompt types:**

| Type | Approx Input Tokens | Prompt |
|------|---------------------|--------|
| Short | ~20 tokens | `"Explain what a neural network is in one sentence."` |
| Medium | ~80 tokens | `"You are a helpful assistant. A user asks: 'I have a dataset of 10,000 customer reviews. What are three practical machine learning approaches I could use to automatically classify them as positive, negative, or neutral? Briefly explain each approach and its trade-offs.'"` |
| Long | ~400 tokens | *(Full passage — see Appendix A)* |

**Protocol:**
1. Load model (Ollama pull/run first time to warm weights)
2. Run **1 discarded warm-up call** per model (not recorded)
3. Run **8 measured calls** per prompt type
4. Record: tok/s, TTFT (ms), total tokens generated
5. Report: **mean ± stdev**, min, max
6. 5-second pause between runs; 15-second cooldown between prompt types
7. Capture `nvidia-smi --query-gpu=memory.used --format=csv,noheader` mid-inference for actual VRAM

**Decision rule:** If stdev > 10% of mean, flag as unstable and note in report.

---

### 2. Quality Benchmark

**Goal:** Measure reasoning quality across quantization formats with enough samples to support statistical claims.

**Parameters (fixed):**
- `temperature: 0.7` (sampling enabled to expose stability)
- `seed: 42`
- `think: false` (non-thinking mode, consistent with Note 003)
- `num_predict: 500`

**Question set (10 questions per category):**

#### Category A: Mathematics (avoid training-contaminated classics)
| Q# | Question | Expected Answer |
|----|----------|-----------------|
| A1 | A car park has 3 floors. Floor 1 has 48 spaces, floor 2 has 35 spaces, floor 3 has 27 spaces. If 60% are occupied, how many spaces are free? | 44 |
| A2 | A rectangle has perimeter 56 cm. Its length is 3 times its width. What is the area? | 147 cm² |
| A3 | If 7 workers complete a task in 12 days, how many days would 4 workers take? | 21 days |
| A4 | A shop discounts a $240 item by 15%, then applies an additional 10% off the discounted price. What is the final price? | $183.60 |
| A5 | What is the sum of all odd numbers from 1 to 99? | 2500 |

#### Category B: Logical Reasoning (novel scenarios)
| Q# | Question | Expected Answer |
|----|----------|-----------------|
| B1 | Five people sit in a row. Anna is not at either end. Bob sits immediately to the left of Carol. Dave is at the far left. Eve is not next to Anna. What is the order? | Dave, Anna, Bob, Carol, Eve |
| B2 | You have 12 balls, one is heavier. Using a balance scale with only 3 weighings, how do you find the heavy one? | Weigh 4 vs 4; if balanced, the heavy ball is in remaining 4. Weigh 2 vs 2 from that group; if balanced, weigh remaining 2. Each step halves candidates. |
| B3 | A light switch controls one of three lamps in the next room. You can flip switches before entering, but can only enter once. How do you identify which switch controls which lamp? | Turn on switch 1 for 5+ minutes, turn off, turn on switch 2, enter room. Hot+off = switch 1; on = switch 2; cold+off = switch 3. |
| B4 | If all Bloops are Razzles, and all Razzles are Lazzles, are all Bloops definitely Lazzles? | Yes (transitive property of set inclusion) |
| B5 | A man builds a house with all 4 sides facing south. A bear walks past. What colour is the bear? | White (polar bear — only possible at North Pole) |

#### Category C: Coding Task (quantization damage often appears here)
| Q# | Task | Evaluation Criteria |
|----|------|---------------------|
| C1 | Write a Python function that returns the nth Fibonacci number using memoization. | Correct output for n=0,1,10,20; uses memoization |
| C2 | Write a Python function to check if a string is a valid IPv4 address. | Handles edge cases: leading zeros, out-of-range octets, wrong format |
| C3 | Given a list of integers, return the two numbers that add up to a target sum (assume one solution exists). | O(n) solution using hash map preferred |

**Scoring:**
- Mathematics: Binary correct/incorrect
- Logic: Correct answer = 1, partially correct reasoning = 0.5, wrong = 0
- Coding: Auto-tested against unit tests (see Appendix B)

**Runs per question:** 3 runs × fixed seed, temperature 0.7 → report modal answer and consistency rate

---

### 3. VRAM Measurement

During each benchmark run, capture actual VRAM usage mid-inference:

```bash
# Background monitor during ollama run
nvidia-smi --query-gpu=timestamp,memory.used,memory.total,utilization.gpu \
  --format=csv --loop-ms=500 > /tmp/vram_log_${model}.csv &
NVIDIA_PID=$!
# ... run benchmark ...
kill $NVIDIA_PID
```

Report: peak VRAM used (not file size estimate).

---

## Appendix A: Long Prompt Text (~400 tokens)

```
You are a technical writer. The following is a description of a distributed database system:

"Our system uses a leader-follower replication model where write operations are sent to the leader node, which then propagates changes to follower nodes asynchronously. We have 3 follower nodes distributed across two geographic regions (Asia-Pacific and Europe). The leader is located in Asia-Pacific. Each follower applies a write-ahead log (WAL) to maintain consistency.

Current challenges we face:
1. Follower nodes in Europe experience 180-200ms replication lag due to network latency
2. During peak hours (09:00-11:00 SGT), the leader handles approximately 4,500 write operations per second
3. Two follower nodes failed simultaneously last month, requiring manual intervention to restore from snapshot
4. Read scalability has become a bottleneck as product teams increasingly bypass the follower-read pattern and query the leader directly

We are evaluating whether to migrate to a multi-leader or leaderless replication model."

Based on the description above, write a structured technical analysis covering:
1. Root causes of each challenge listed
2. Pros and cons of migrating to multi-leader replication
3. Pros and cons of migrating to leaderless replication (Dynamo-style)
4. Your recommendation with justification
```

---

## Appendix B: Coding Task Unit Tests

```python
# C1: Fibonacci
assert fib(0) == 0
assert fib(1) == 1
assert fib(10) == 55
assert fib(20) == 6765

# C2: IPv4 Validator
assert is_valid_ipv4("192.168.1.1") == True
assert is_valid_ipv4("256.0.0.1") == False
assert is_valid_ipv4("192.168.01.1") == False  # leading zero
assert is_valid_ipv4("192.168.1") == False
assert is_valid_ipv4("192.168.1.1.1") == False
assert is_valid_ipv4("abc.def.ghi.jkl") == False

# C3: Two Sum
assert two_sum([2, 7, 11, 15], 9) in [[0, 1], [1, 0]]
assert two_sum([3, 2, 4], 6) in [[1, 2], [2, 1]]
assert two_sum([3, 3], 6) in [[0, 1], [1, 0]]
```

---

## Reporting Template (Note 006)

Note 006 will follow this structure:

1. Executive Summary
2. Speed Results — table with mean ± stdev, min, max per model/prompt type
3. VRAM — actual measured values per model
4. Quality Results — per category, per model; accuracy %, consistency rate
5. Comparison vs Note 003 — what changed, what improved
6. Known Remaining Limitations
7. Conclusions & Updated Recommendations

---

## Reproduction Steps

```bash
# SSH into GX10
ssh denniswong@192.168.1.5

# Run benchmark script
bash ~/benchmark/run_benchmark.sh

# Results saved to:
# ~/benchmark/results/speed_results.csv
# ~/benchmark/results/quality_results.json
# ~/benchmark/results/vram_log_*.csv
```

---

*→ [Note 006 — Benchmark Results](note-006.md)*
*← [Back to Index](../README.md)*
