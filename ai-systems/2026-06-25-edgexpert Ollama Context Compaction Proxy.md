# edgexpert Ollama Context Compaction Proxy

**日期：** 2026-06-25
**機器：** `[[SSH_TARGET_3060ae]]`（NVIDIA DGX Spark GB10）
**結論：透明 proxy 在 :8002，自動壓縮過長 context，RD 只改一個 URL**

---

## 背景

Sonar Agent（AI agentic workflow）執行長任務時，多輪工具呼叫累積的 message history 會超過 Ollama 的 context window，導致回應被截斷（`stopReason: "length"`）。

升 context window（16k→65k，見 `edgexpert-ollama-context-upgrade.md`）是第一道防線，但長任務仍可能耗盡。Server 端自動壓縮是第二道防線，且 RD 的程式碼完全不需要修改。

---

## 架構

```
RD 自動化腳本 / Sonar Agent
    ↓ :8002
Compact Proxy（ollama_compact_proxy.py）
    ├─ 檢查 messages 估算 token 數
    ├─ 超過 60% → 先壓縮再轉
    └─ 未超過 → 直接轉
         ↓ :8000
    Ollama（gpt-oss-fast / gemma4-fast）

open-webui（短對話）
    ↓ :8000（直連，不過 proxy）
    Ollama
```

**兩個 port 並存，互不干擾：**

| Port | 用途 | 壓縮 |
|------|------|------|
| :8000 | Ollama 直連（open-webui、一般測試） | 無 |
| :8002 | Compact Proxy（RD 自動化、agentic workflow） | 自動 |

---

## 壓縮邏輯

```
每個 /v1/chat/completions 請求進來時：
  估算 token 數 = 所有 message content 總字數 ÷ 3.5
  if 估算 > num_ctx × 60%：
    保留 system messages
    保留最近 6 條 messages（3 輪）
    舊的 messages → 呼叫同一個 model 做摘要（150字）
    用摘要取代舊訊息
  轉發給 Ollama :8000
```

各模型 context 設定（觸發壓縮的 token 數）：

| 模型 | num_ctx | 壓縮門檻（60%） |
|------|---------|----------------|
| gpt-oss-fast:latest | 65535 | ~39000 tokens |
| gemma4-fast:latest | 32768 | ~19600 tokens |

---

## 部署資訊

- **Script：** `[[LINUX_PATH_c1378b]]
- **Service：** `ollama-compact-proxy.service`（systemd，開機自啟）
- **Log：** `sudo journalctl -u ollama-compact-proxy -f`

```bash
# 狀態確認
sudo systemctl status ollama-compact-proxy

# 重啟
sudo systemctl restart ollama-compact-proxy

# 即時 log
sudo journalctl -u ollama-compact-proxy -f
```

---

## RD 使用方式

只需把 `base_url` 改一個 port：

```python
# 改前
client = OpenAI(base_url="http://[[PRIVATE_IP_f1cc91]]:8000/v1", api_key="dummy")

# 改後（自動壓縮）
client = OpenAI(base_url="http://[[PRIVATE_IP_f1cc91]]:8002/v1", api_key="dummy")
```

其他程式碼完全不變。streaming、tool calling、`/v1/models` 等端點全部透明轉發。

---

## 踩坑

- **port 8001 已被 `edgexpert_api_logger.py` 佔用** → 改用 8002
- **nohup 啟動的 orphan process 佔住 port** → `pkill -f ollama_compact_proxy` 後才能讓 systemd 接管
- **fastapi/uvicorn 裝在 `~/.local/bin`** → service 的 `Environment=PATH` 要加 `[[LINUX_PATH_5aba78]]
- **token 估算用字元數 ÷ 3.5**（中英混合粗估），不是精確值，偏保守沒關係

---

## 未來可優化

- 壓縮門檻可以從環境變數設定（現在 hardcode 60%）
- 壓縮摘要可以加入「本輪任務目標」避免摘要遺漏重點
- 若 Ollama 回傳實際 `usage.prompt_tokens`，可改用精確 token 數觸發
