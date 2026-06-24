# LLM 與 AI 論文精選清單 (Self-Study Guide)

這份清單彙整了從 Transformer 原始論文到 2025 年最前沿的推理與生成技術，適合用於 AI 與 LLM 的深度學習與研究。

---

# 一、基礎模型與架構 (Foundation Models & Architecture)

## 1.1 核心架構論文

### Transformer 原始論文
- **Attention Is All You Need** (Google, 2017)
  提出 Transformer 架構，完全基於注意力機制，拋棄 RNN 與 CNN。這是現代所有 LLM 的基礎架構，在機器翻譯任務上達到 SOTA，並開創了預訓練語言模型的新時代。**必讀經典。**

### BERT
- **BERT: Pre-training of Deep Bidirectional Transformers for Language Understanding** (Google, 2018)
  提出雙向 Transformer 預訓練方法。開創了「預訓練-微調」範式。**必讀經典。**

### RoBERTa
- **RoBERTa: A Robustly Optimized BERT Pretraining Approach** (Meta, 2019)
  系統性研究 BERT 的超參數優化，揭示了預訓練細節的重要性。

### GPT 系列
- **Language Models are Unsupervised Multitask Learners** (GPT-2, OpenAI, 2019)
  展示了大規模語言模型的無監督多任務學習能力。
- **Language Models are Few-Shot Learners** (GPT-3, OpenAI, 2020)
  175B 參數展示了驚人的 few-shot 學習能力，開啟了大模型時代。**必讀經典。**

### PaLM
- **PaLM: Scaling Language Modeling with Pathways** (Google, 2022)
  展示了極致規模化的能力邊界，在多步推理任務上實現突破。

### Llama 系列
- **Llama 2: Open Foundation and Fine-Tuned Chat Models** (Meta, 2023)
  開源 LLM 的里程碑，推動了開源生態發展。**必讀。**

## 1.2 Scaling Laws 與訓練策略

### 縮放定律 (Scaling Laws)
- **Scaling Laws for Neural Language Models** (OpenAI, 2020)
  揭示模型性能與大小、數據量、計算量之間的冪律關係。
- **Training Compute-Optimal Large Language Models (Chinchilla)** (DeepMind, 2022)
  提出 Chinchilla 最優訓練原則，改變了業界對模型訓練的認知。

---

# 二、高效架構 (Efficient Architectures)

## 2.1 Mixture-of-Experts (MoE)

### 開創性工作
- **Outrageously Large Neural Networks** (Google, 2017)
  提出稀疏門控 MoE 層，實現了模型容量的巨大提升。
- **GShard** (Google, 2020)
  實現 600B+ 參數的多語言翻譯模型訓練。

### Switch Transformer
- **Switch Transformers** (Google, 2021)
  簡化 MoE 路由算法，實現萬億參數稀疏模型。

### DeepSeek 系列
- **DeepSeekMoE** (DeepSeek, 2024)
  提出細粒度專家分割和共享專家策略。
- **DeepSeek-V2** (DeepSeek, 2024)
  引入 Multi-head Latent Attention (MLA) 壓縮 KV cache，效率大幅提升。
- **DeepSeek-V3 Technical Report** (DeepSeek, 2024)
  當前最重要的開源 MoE 模型之一。

### Mixtral
- **Mixtral of Experts** (Mistral AI, 2024)
  8x7B 稀疏 MoE 架構，性能表現卓越。

## 2.2 注意力機制優化 (Attention Optimization)

### FlashAttention
- **FlashAttention: Fast and Memory-Efficient Exact Attention** (Stanford, 2022)
  將注意力計算記憶體從 O(N²) 降至 O(N)。**現代 LLM 訓練必備。**
- **FlashAttention-2** (Tri Dao, 2023)
  進一步優化，達到理論峰值 FLOPS 的 50-73%。

### 多頭注意力變體
- **Multi-Query Attention (MQA)** (Google, 2019)
  所有頭共享 K/V，減少推理時記憶體帶寬需求。
- **Grouped-Query Attention (GQA)** (Google, 2023)
  在 MHA 和 MQA 之間取得平衡，已被 Llama 2 等主流模型採用。

### 高效推理服務 (vLLM)
- **PagedAttention (vLLM)** (UC Berkeley, 2023)
  借鑒作業系統分頁技術管理 KV cache，吞吐量提升 2-4x。

### 長序列注意力 (Long Sequence)
* **Sparse Transformers** (OpenAI, 2019)
* **Longformer** (AllenAI, 2020)
* **Big Bird** (Google, 2020)
* **Ring Attention** (UC Berkeley, 2023)
* **StreamingLLM** (MIT, 2023)

## 2.3 位置編碼 (Positional Encoding)

* **RoPE (RoFormer)** (2021)
  旋轉位置編碼，現代 LLM 的標準。
* **ALiBi** (2021)
  線性偏置，支援長度外推。

## 2.4 正規化與 FFN 改進
* **Pre-LN Transformer** (2020)
* **RMSNorm** (2019)
* **GLU Variants (SwiGLU, GeGLU)** (2020)

## 2.5 替代架構 (Alternative Architectures)

### SSM (State Space Models)
* **Mamba** (CMU/Princeton, 2023)
  線性時間複雜度，Transformer 的有力挑戰者。
* **RWKV** (2023)
  結合 Transformer 的訓練效率與 RNN 的推理複雜度。

### Diffusion Language Models
* **Diffusion-LM** (Stanford, 2022)
* **DiffuSeq** (2022)
* **MDLM** (Cornell, 2024)

---

# 三、模型對齊與人類反饋 (Alignment & RLHF)

## 3.1 RLHF 方法
* **InstructGPT** (OpenAI, 2022): ChatGPT 的技術基礎，實現了 SFT $\rightarrow$ RM $\rightarrow$ PPO。
* **Constitutional AI** (Anthropic, 2022): 使用 RLAIF (AI 反饋) 實現安全性。

## 3.2 簡化對齊 (DPO)
* **DPO (Direct Preference Optimization)** (Stanford, 2023)
  無需訓練 Reward Model，直接優化策略，更穩定高效。

## 3.3 Reasoning RL (推理強化)
* **DeepSeekMath (GRPO)** (Shao et al., 2024)
  提出 GRPO (Group Relative Policy Optimization)，降低 PPO 成本。

## 3.4 過程監督 (Process Supervision)
* **Let's Verify Step by Step** (Lightman et al., 2023)
  強調「逐步過程標註」比「只看最後答案」更有效。

---

# 四、監督微調 (SFT) & 知識蒸餾

* **FLAN** (Wei et al., 2021)
* **Self-Instruct** (Wang et al., 2022)
* **LIMA** (Zhou et al., 2023): 「資料品質 >> 數量」的代表作。
* **Distilling Step-by-Step**: 透過 Rationale (推理步驟) 進行知識蒸餾。

---

# 五、檢索增強生成 (RAG)
* **RAG (Meta, 2020)**: 結合預訓練參數與非參數記憶的基礎架構。

---

# 六、提示工程、推理與推理時計算 (Inference-Time Compute)

## 7.1 思維鏈 (CoT)
* **Chain-of-Thought Prompting** (Google, 2022): 激發 LLM 的複雜推理能力。

## 7.2 推理路徑與搜尋
* **Self-Consistency** (2022): 多路徑多樣化採樣。
* **Tree of Thoughts (ToT)** (2023): 將推理形式化為樹搜尋。
* **ReAct** (2022): 推理 (Thought) 與行動 (Act) 的交錯。
* **Reflexion** (2023): 語言 Agent 的自我反思機制。

## 7.3 推理時縮放 (Test-Time Compute)
* **Scaling LLM Test-Time Compute Optimally** (Snell et al., 2024): 討論推理預算與模型參數的 Trade-off。

---

# 七、參數高效微調 (PEFT)
* **LoRA** (Microsoft, 2021)
* **QLoRA** (UW, 2023): 4-bit 量化 + LoRA。

---

# 八、量化與加速 (Quantization & Efficiency)

* **Speculative Decoding (Speculative Sampling)**: 使用 Draft model 加速。
* **Medusa**: 多頭預測加速。
* **vLLM (PagedAttention)**: 高效記憶體管理。

---

# 九、Diffusion Transformers (DiT) 與視覺生成

## 9.1 生成模型數學基礎
* **Neural ODE / Normalizing Flows / Flow Matching / Rectified Flow**

## 9.2 DiT 架構與發展
* **DiT (Scalable Diffusion Models with Transformers)** (Peebles & Xie, 2023)
* **U-ViT** (2022)
* **PixArt-α / PixArt-Σ** (2023/2024)

## 9.3 MMDiT (多模態 DiT)
* **Stable Diffusion 3 / SD3** (Esser et al., 2024): 引入 MMDiT 雙模態 Transformer。
* **Sora (OpenAI, 2024)**: 影片生成作為世界模擬器。

---

# 十、AI Agent 與推理 (AI Agents & Reasoning)

* **Reasoning RL**: DeepSeek-R1/o1 類型的技術演進。
* **Agentic Workflow**: 結合 Skill/Tool 的單 Agent vs 多 Agent 系統。

---

# 十一、綜合資源與教程 (Resources & Tutorials)

* **Sebastian Raschka 系列文章**: 深入分析最新 LLM 架構與推理模型。
