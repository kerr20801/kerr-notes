# Home Lab AI vs AI：Red Team AI 自主掃描，Blue Team AI 管基礎設施

## 架構

```
OpenClaw（隔離 VM）          C.K（基礎設施 AI）
  外部視角，不知道內部架構       知道所有服務在哪、有真實操作權限
  自主執行掃描工具              管 K8S / NGINX / ELK
  Nemotron 120B（外部 API）    gpt-oss-20B（本地 Ollama）
         ↓                            ↑
         └─────── 人類審計 ───────────┘
                  看發現什麼
                  決定要不要讓 C.K 處理
```

兩個 AI 在同一個 home lab 跑，角色完全分離。OpenClaw 只看得到對外暴露的介面，C.K 知道內部所有細節但不主動攻擊。人在中間審計。

---

## 關鍵設計決策

### Red Team AI 必須用外部模型，本地模型不行

第一個嘗試：讓本地 gpt-oss-20B（MoE 推理模型）當 Red Team AI。

結果：agent framework 永遠不執行工具。原因是推理模型的輸出格式：

```
<thinking>
We should call bash with red_team_ai.py...
tool_call: bash("python3 red_team_ai.py scan ...")
</thinking>
{"content": "我已安排掃描，請稍候..."}
```

工具呼叫埋在 thinking block 裡，agent framework 只看 `content`，永遠看不到 tool call。
模型以為自己在執行，其實什麼都沒發生。

換 OpenRouter Nemotron 120B 之後正常——標準 function calling，agent 能正確解析。

**結論**：推理模型的 thinking prefix 會讓 tool use 失效，需要用支援標準 function calling 的模型。

### Red Team 隔離在獨立 VM

不是同一台機器跑兩個 process，是真正的獨立 VM：
- 對 Red Team 來說，目標服務就是黑箱
- 就算 Red Team AI 被 prompt injection 或做了奇怪的事，影響範圍限制在這台 VM
- 壞了 re-clone 就好，10 分鐘重建

### Approval allowlist 控制自主執行範圍

agent framework 預設不允許 shell 執行。明確 allowlist 比完全放開安全：

```bash
openclaw approvals allowlist add --agent main 'bash *'
openclaw approvals allowlist add --agent main 'python3 *'
```

沒在 allowlist 的指令還是會要求人工確認。

---

## 建置踩過的坑

### 1. Gateway 只綁 loopback

`openclaw gateway` 預設只監聽 `127.0.0.1`。
正確指令是 `--bind lan`，不是 `--bind 0.0.0.0`（那個參數無效）。

### 2. FortiGate 擋 GitHub Models 後端

GitHub Models API 走 `models.inference.ai.azure.com`，被 FortiGate 預設封鎖。

解法：在另一台機器裝 squid proxy，Red Team VM 的 systemd service 加：

```
Environment=https_proxy=http://proxy-host:3128
```

squid 只 allow Red Team VM 的 IP，不開放全網段。

### 3. 免費 API 限制

| Provider | 問題 |
|----------|------|
| GitHub Models | 免費版 8K token 上限，比 system prompt 還小 |
| Cerebras | 免費額度快速耗盡，rate limit 429 |
| OpenRouter Nemotron | 免費無限制，120B 夠用 ✅ |

---

## 驗證：AI 自主執行 Red Team 掃描

```
指令：「請執行 red_team_ai.py scan https://target-url 並回報結果」

OpenClaw 自動：
  1. 執行 bash 指令
  2. 等待掃描完成（~30 秒）
  3. 整理成繁體中文摘要
  4. 回報結果
```

不需要每個步驟都下指令，一句話到結果。

---

## 這個設計解決什麼問題

手動每次掃描很麻煩，而且容易忘。有了這個設置：
- 改完 NGINX 規則 → 叫 OpenClaw 掃一次
- 部署新服務 → 叫 OpenClaw 從外部看有沒有暴露不該暴露的東西
- C.K 修好之後 → 再掃一次確認

兩個 AI 的分工讓這個流程可以不依賴人工記憶。
