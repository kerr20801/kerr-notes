# edgexpert GB10 Ollama Context Window 升級記錄

**日期：** 2026-06-25
**機器：** `[[SSH_TARGET_3060ae]]`（NVIDIA DGX Spark GB10，128GB 統一記憶體）
**結論：gpt-oss-fast + gemma4-fast 均從 16384 升至 32768，記憶體穩定在 ~100GB**

---

## 背景

Sonar Agent（threat-sonar-legacy 專案的 AI Agent，model: `gpt-oss-fast:latest`）在執行多步驟調查任務時，跑了 25+ 輪工具呼叫後對話歷史累積至上限，回應被截斷在句子中間：

```
stopReason: "length"
input: 15555, output: 829, total: 16384  ← 剛好撞到 context 上限
```

根因：`OLLAMA_CONTEXT_LENGTH=16384`（server 全域設定），agentic workflow 的多輪 tool result 累積會快速消耗 context window，而非單次對話用量不足。

---

## 診斷

這與另一個問題（RD 的 max_tokens 太小導致 content 空白）是**不同問題**：

| 問題 | 原因 | 修法 |
|------|------|------|
| content 空白（RD 端） | max_tokens 太小（如 512），reasoning phase 先吃光 | API call 用 max_tokens >= 4096 |
| 回應截斷（Sonar Agent） | context window 滿（16k tokens 跨多輪累積） | 升 num_ctx |

---

## 修改方式

不動全域 `OLLAMA_CONTEXT_LENGTH`（避免影響所有模型），改各別 Modelfile 的 `num_ctx` 參數：

```bash
# 取出 Modelfile
OLLAMA_HOST=localhost:8000 ollama show gpt-oss-fast:latest --modelfile > /tmp/gpt-oss-fast.modelfile

# 把 FROM blob 路徑改成 tag（避免 permission denied）
sed -i 's|FROM /usr/share/ollama/.*|FROM gpt-oss-fast:latest|' /tmp/gpt-oss-fast.modelfile

# 升 num_ctx
sed -i 's/PARAMETER num_ctx 16384/PARAMETER num_ctx 32768/' /tmp/gpt-oss-fast.modelfile

# 重建（不重下載，只更新 manifest）
OLLAMA_HOST=localhost:8000 ollama create gpt-oss-fast:latest -f /tmp/gpt-oss-fast.modelfile

# gemma4-fast 同上（略）

# 重啟套用
sudo systemctl restart ollama
```

---

## 記憶體影響

GB10 128GB 統一記憶體，兩個模型均常駐（`OLLAMA_KEEP_ALIVE=-1`）：

| 項目 | 估算 |
|------|------|
| gpt-oss-fast 模型權重 | ~65GB |
| gemma4-fast 模型權重 | ~19GB |
| OS + process overhead | ~8GB |
| **KV cache（升前 16k × 3 slots × 2 模型）** | ~8GB |
| **KV cache（升後 32k × 3 slots × 2 模型）** | ~16GB |

升前 htop：~106GB；升後 htop：~100GB 出頭（兩個模型重新 load 後觀察穩定）。

KV cache 與 context 線性正比，升一倍 context ≈ KV cache 多 8GB，128GB 範圍內可承受。

---

## 現狀

| 模型 | num_ctx | num_predict | num_parallel |
|------|---------|-------------|--------------|
| gpt-oss-fast:latest | **32768** | 4096 | 3 |
| gemma4-fast:latest | **32768** | 4096 | 3 |

Sonar Agent 長 session 現在可跑到 ~32k tokens 才截斷，比原本多一倍。

---

## 注意事項

- **大量並發才能真正壓測**：目前觀察記憶體穩定，但 3 slots × 2 模型同時高負載時才是真實考驗
- **若記憶體不足**：優先把 `num_parallel` 從 3 降到 2（比降 context 影響小）
- **更根本的解法**：Agent framework 做 context compaction（把舊 tool result 摘要壓縮），不依賴無限擴大 context window

---

## 踩坑

- `ollama create` 時 FROM 用 blob 路徑 → `permission denied`（ollama user 持有 blob，kerr 無法讀）
  → 改成 `FROM gpt-oss-fast:latest`（tag 引用），Ollama API 直接用已有的 layer，正常
- `ollama show --modelfile` 預設連 `localhost:11434`，需加 `OLLAMA_HOST=localhost:8000`
