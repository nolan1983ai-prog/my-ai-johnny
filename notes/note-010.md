# Note 010 — Cisco Live Protect & Tetragon: 從產品研究到動手實驗

**Date:** 2026-08-31  
**Topic:** Cisco Live Protect（NX-OS 即時 CVE 防護）深度研究 + 家用 Linux AI 主機上的 Tetragon 實驗計劃

## Summary

研究 Cisco Live Protect（基於 eBPF/Tetragon 的 NX-OS 即時漏洞防護），由產品定位、運作原理、CLI 實務，到在自家 Linux AI 主機（ASUS Ascent GX10）上部署 upstream Tetragon 的實驗計劃。目標：親手做過 kernel-level security enforcement，用第一身經驗理解 Cisco 將開源技術產品化的路徑。

---

## Part 1 — Cisco Live Protect 研究筆記

### 解決咩問題？

傳統 CVE patching 需要 maintenance window、重啟、downtime。業界普遍嘅痛點係「patching gap」：攻擊者由 CVE 公佈到利用只需要幾小時，而防禦方喺生產環境測試同部署 patch 往往需要幾星期甚至幾個月。AI data center 對 availability 要求極高，令呢個 gap 更加致命。

### Live Protect 係咩？

- **eBPF + enterprise-grade Tetragon agent 直接內置 NX-OS**（10.6(1)F 起）— Cisco 稱為 market first
- 核心賣點：**零 downtime 套用 vulnerability shield** — 唔使 reboot 即時擋攻擊，正式 PSIRT patch 留返標準 maintenance window 做
- Shield = CVE compensating controls；做咗 PSIRT bundle upgrade 後會自動移除

### 四大防護範圍

1. Privilege escalation 防護
2. Control-plane / routing plane 保護
3. API / CLI security
4. File I/O monitoring

### 運作細節（官方 Security Configuration Guide）

- NX-OS 上以 `feature nxsecure` 開啟；Tetragon container 喺 Apphosting Framework 上運行（需 ≥2GB bootflash）
- Shield 以簽名 package（`.lps`）形式由 cisco.com 下載，用 `nxsecure policy add` 安裝
- 兩個 mode：**Monitor**（觀察出 JSON event log）→ **Enforce**（kernel 層即時攔截）
- 官方建議 workflow：先 monitor 行一排 → 檢查 `show nxsecure policy status` 嘅 hit counts → 無誤傷先切 enforce
- Event logs 係 JSON 格式（含 PID、UID、binary、arguments、parent process chain），可經 telemetry（sensor path `event-nxsecure`）export 去 Splunk / SIEM
- 10.6(3)F 起支援 syslog 同 per-policy log history

### 平台支援（重點：唔使買新 hardware！）

- 支援普通 Nexus 9000 — 包括較舊嘅 N9300-FX / FX2 / FX3 / GX / GX2 / H1 / H2R（≥24GB RAM），N9200、N9400、N9100 系列
- 呢個係純軟件方案：eBPF agent 跑喺 switch 嘅 Linux kernel，同 DPU 無關
- License 要求：NXOS_ESSENTIALS 或以上
- 限制：同 app-hosting、dockerbox、config replace、auditD 唔相容；CPU 影響官方稱 negligible

### 同 Hypershield 嘅分別（常見混淆）

| | Live Protect | Hypershield |
|---|---|---|
| 定位 | OS 層 CVE 即時防護 | 網絡流量 segmentation（Hybrid Mesh Firewall） |
| 引擎 | Tetragon eBPF agent 喺 NX-OS kernel | Smart Switch DPU 上嘅 enforcement |
| 硬件要求 | 普通 N9000 就得 | 要 Smart Switch（DPU） |
| 防護對象 | Exploit / privilege escalation | East-west lateral movement |

兩者同源（Isovalent eBPF 技術血統），但係**獨立產品，唔使綁埋一齊買**。

### Tetragon 小知識

- 開源項目（github.com/cilium/tetragon），由 Isovalent 開發 — 即 Cilium 背後嘅公司，2024 年被 Cisco 收購
- Upstream 嘅概念叫 **TracingPolicy**（YAML 格式）；「shield」/`.lps` 係 Cisco 將佢產品化後嘅包裝術語
- Cisco 嵌入 NX-OS 嘅係企業級 build：整合、測試、簽名，隨 NX-OS image 出貨 — agent 生命週期由 NX-OS release 管理，客戶唔需要獨立維護

### 點解開源都難以複製（三道牆）

1. **Network OS 封閉**：競爭對手要自行將 eBPF agent 整合入自己嘅 NOS，係以年計工程
2. **Isovalent 團隊已屬 Cisco**：有 upstream code 唔等於有原作者做整合優化
3. **Pipeline 先係真壁壘**：由 CVE 評估 → 寫 policy → 真機測試 → 簽名 → 發佈 → support，呢條生產線冇捷徑

一句總結：eBPF 技術係開源嘅，但「廠商將 CVE 變成簽名 policy 即推落 switch」呢件事，目前只有 Cisco 做。

---

## Part 2 — Tetragon 動手實驗計劃（GX10 Lab）

### 環境

- ASUS Ascent GX10（NVIDIA GB10 Grace Blackwell）
- Ubuntu 24.04.4 LTS，kernel 6.17.0-1018-nvidia
- 已運行 Ollama（本地 LLM inference 服務）
- Tetragon 以 Docker 方式部署（`--privileged --pidns=host --cgroupns=host`），唔影響現有 workload

### Test Cases（由淺入深）

**TC1：Process 生命週期觀察（熱身）**
Observer mode + `tetra getevents`，實時觀察 process_exec events（binary、PID、parent chain、arguments、UID）。驗證安裝 + 認住 JSON event 格式（同 NX-OS Live Protect 文檔同款）。

**TC2：Privilege Escalation 偵測**
TracingPolicy 監察 sudo / setuid / credentials 變化；測試 `sudo -i` 同模擬非 root 寫 /etc/passwd。呢類正係 Live Protect shield 主要防護嘅 CVE 類型。

**TC3：敏感檔案存取監察**
Policy 針對 `/etc/shadow`、`~/.ssh/`、agent secrets 目錄。對應自己環境嘅真威脅面 — secrets 外洩監察。

**TC4：守護 Ollama（貼身應用）**
監察接觸 ollama process / port 11434 嘅連線同 binary 執行；正常 curl `/api/generate` 做對照。將 runtime security 綁落真正嘅 AI workload。

**TC5：Enforce Mode 即時攔截（殺手鐧）**
Policy「任何 process 執行 `/tmp/fake-malware` 一律 SIGKILL」；寫個無害 script 測試。親身體驗 kernel 層 enforcement — monitor→enforce 嗰一刻就係 Live Protect 嘅精髓。

**TC6：Network Observability**
用 `tetra trace` 追蹤對外連線嘅 process ↔ flow 對應（邊個 process 連咗邊個 IP:port）。對應 Smart Switch 嘅 network-to-process visibility 賣點。

**TC7（畢業作）：模擬 CVE Shield 完整流程**
揀一個已知 Linux 濫用手法，寫針對性 policy：先 monitor → 確認命中（check hits）→ 切 enforce。一比一重演 NX-OS 上 `.lps` shield 嘅生命週期。

### 建議次序

TC1 → TC3 → TC5（核心路線 ~1.5 小時）→ TC2/TC4/TC6 → TC7

### 安全預告

全程 Docker 容器化跑 Tetragon；policy 只針對測試用假目標（/tmp 檔案），唔會掂 Ollama、Docker daemon 同正常操作。

---

## 學到嘅嘢

1. **開源 → 產品化嘅價值路徑**：Tetragon（開源技術）→ NX-OS 嵌入（整合）→ PSIRT pipeline（營運）→ 簽名 shield（信任）。技術只係第一步，pipeline 先係 moat。
2. **Lab-first 學習法**：研究完產品文檔，最有效嘅深化方法係喺自己機器上重演核心機制 — 由「背書」變成「講親身經歷」。
3. **術語精確性**：shield vs TracingPolicy、Live Protect vs Hypershield — 同 technical audience 溝通，分清 marketing 詞同技術詞好重要。

## 下一步

- [ ] GX10 上面執行 TC1-TC5，記錄實驗結果
- [ ] 比較 upstream Tetragon events 同 NX-OS Live Protect 文檔嘅 JSON 格式
- [ ] TC7 完成後寫總結（可能有 note-011）

## 參考

- Cisco Live Protect Solution Overview（cisco.com 公開文檔）
- Cisco Nexus 9000 NX-OS Security Configuration Guide 10.6(x) — Secure NX-OS with Live Protect
- Tetragon 開源項目：https://github.com/cilium/tetragon
- Tetragon 官方文檔：https://tetragon.io
