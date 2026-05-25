# Note 007 — llama.cpp vs Ollama: Installation, Trade-offs, and Why Qwopus Needs llama.cpp

**Date:** 2026-05-25
**Hardware:** Asus Ascent GX10 / NVIDIA GB10 Blackwell / 121.6 GiB VRAM
**Topic:** llama.cpp installation on GB10, comparison with Ollama, and rationale for using llama.cpp with Qwopus3.6-27B-v2-MTP

---

## Summary

This note documents the full llama.cpp installation process on the GX10, explains the architectural differences between llama.cpp and Ollama, and explains why Qwopus3.6-27B-v2-MTP required llama.cpp rather than Ollama for correct deployment.

---

## 1. What is llama.cpp?

llama.cpp is the **raw inference engine** — a C++ library and server that runs GGUF-format LLMs directly on CPU or GPU. It is the foundational layer on which many tools (including Ollama) are built.

## 2. What is Ollama?

Ollama is a **user-friendly wrapper around llama.cpp**. It handles model downloading, management, and serving with a simple CLI and API, but abstracts away many low-level configuration options.

---

## 3. llama.cpp vs Ollama — Pros and Cons

| | **llama.cpp** | **Ollama** |
|---|---|---|
| **Setup** | Manual: clone, cmake, compile | One-line install script |
| **GPU arch control** | Full control (`-DCMAKE_CUDA_ARCHITECTURES=121a`) | Automatic, may not be optimal |
| **Model management** | Manual `.gguf` download | `ollama pull <model>` |
| **Parameters** | Full control (flash-attn, ctx-size, reasoning-format, etc.) | Limited options |
| **MTP / speculative decoding** | Supported natively | Not supported |
| **Performance ceiling** | Maximum (hand-tuned) | Slightly lower (abstracted) |
| **API** | OpenAI-compatible `/v1` | OpenAI-compatible `/v1` |
| **Best for** | Advanced users, benchmarking, custom models | Quick setup, daily use, model exploration |

---

## 4. llama.cpp Installation on GX10 (GB10 Blackwell)

### Prerequisites
- CUDA 13.0 (`/usr/local/cuda/bin/nvcc`)
- cmake, gcc 13.3
- Ubuntu 24.04 aarch64

### Step-by-step

```bash
# 1. Clone llama.cpp
git clone --depth=1 https://github.com/ggml-org/llama.cpp ~/llama.cpp
cd ~/llama.cpp

# 2. Configure with CUDA for Blackwell (sm_121a)
export PATH=/usr/local/cuda/bin:$PATH
cmake -B build \
  -DGGML_CUDA=ON \
  -DCMAKE_CUDA_ARCHITECTURES=121

# cmake auto-upgrades 121 → 121a for Blackwell
# Output: Using CMAKE_CUDA_ARCHITECTURES=121a

# 3. Build (8 parallel jobs, ~5-10 min)
cmake --build build --config Release -j8

# 4. Verify
./build/bin/llama-server --version
# version: 1 (549b9d8) — built with GNU 13.3.0 for Linux aarch64
```

### Key cmake flags
| Flag | Value | Why |
|------|-------|-----|
| `DGGML_CUDA` | ON | Enable GPU acceleration |
| `CMAKE_CUDA_ARCHITECTURES` | 121 | GB10 Blackwell compute capability (auto-upgrades to 121a) |

> ⚠️ **Critical:** Without specifying `sm_121a`, llama.cpp falls back to CPU or a generic CUDA build — you lose most GPU acceleration on Blackwell.

---

## 5. Downloading Qwopus3.6-27B-v2-MTP

```bash
# Use hf CLI (not huggingface-cli, which is deprecated)
PATH=$HOME/.local/bin:$PATH

hf download Jackrong/Qwopus3.6-27B-v2-MTP-GGUF \
  Qwopus3.6-27B-v2-MTP-Q4_K_M.gguf \
  --local-dir ~/models/qwopus3.6-mtp

# File size: ~16 GB
```

---

## 6. Starting llama-server

```bash
nohup ~/llama.cpp/build/bin/llama-server \
  -m ~/models/qwopus3.6-mtp/Qwopus3.6-27B-v2-MTP-Q4_K_M.gguf \
  -ngl 999 \
  --flash-attn on \
  --ctx-size 49152 \
  --host 0.0.0.0 \
  --port 8080 \
  --reasoning-format deepseek \
  > ~/models/llama-server.log 2>&1 &
```

Access at `http://localhost:8080` (built-in chat UI) or via OpenAI-compatible API at `/v1`.

---

## 7. Why Qwopus Needs llama.cpp, Not Ollama

Three reasons:

### 7.1 MTP (Multi-Token Prediction) support
Qwopus3.6-27B-v2-MTP uses speculative decoding via MTP to achieve **2.3× faster inference**. Ollama does not support MTP. Using Ollama would silently ignore MTP and run at base speed.

### 7.2 GPU architecture flag
Ollama's bundled llama.cpp binary may not be compiled specifically for `sm_121a` (GB10 Blackwell). Building from source with `-DCMAKE_CUDA_ARCHITECTURES=121` ensures native Blackwell instructions are used.

### 7.3 Model availability
Qwopus is not in the Ollama model registry. It is only available as a GGUF file on Hugging Face (`Jackrong/Qwopus3.6-27B-v2-MTP-GGUF`). Ollama cannot pull it directly.

---

## 8. Connecting llama-server to Open WebUI

Open WebUI supports OpenAI-compatible backends. To add llama-server:

1. Admin Panel → Settings → Connections → Add OpenAI API
2. URL: `http://172.17.0.1:8080/v1` *(use Docker host gateway, not localhost)*
3. API Key: any value (not validated)
4. Connection type: **Local**

> ⚠️ Inside Docker, `localhost` refers to the container itself. Use `172.17.0.1` to reach the host machine.

Model will appear as `Qwopus3.6-27B-v2-MTP-Q4_K_M.gguf` in the model selector.

---

## 9. Performance on GX10

Official benchmark (run on GB10 server, same hardware):

| Metric | Qwen3.6-27B (base) | Qwopus3.6-27B-v2-MTP |
|--------|--------------------|-----------------------|
| Overall tok/s | 6.29 | **10.46** |
| Total eval time | 14,901s | **6,488s** |
| Completion tokens | 93,802 | **67,862** |
| Speed gain | — | **+2.3×** |

---

## Key Takeaways

- llama.cpp = raw engine; Ollama = friendly wrapper around llama.cpp
- For standard models and daily use → Ollama is simpler and sufficient
- For Qwopus MTP, custom GGUF models, or squeezing maximum performance → llama.cpp is necessary
- On Blackwell (GB10), always compile with `-DCMAKE_CUDA_ARCHITECTURES=121` or GPU acceleration is compromised
- When adding llama-server to Open WebUI from Docker, use `172.17.0.1`, not `localhost`
