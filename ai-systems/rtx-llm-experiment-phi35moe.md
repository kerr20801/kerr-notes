# RTX 4070 大模型混搭實驗：Phi-3.5-MoE vs gemma4

**日期：** 2026-06-25
**結論：維持 gemma4，Phi-3.5-MoE 實驗結束**

---

## 背景

RTX 4070（[[PRIVATE_IP_c0a342]]）只有 12GB VRAM，目前跑 Ollama gemma4:26b（streaming answer backend，Argus 使用）。
想探索能否用 llama.cpp 的 VRAM+RAM hybrid（`-ngl N`）在同一台機器跑更強的模型，來提升 Argus 回答品質。

---

## 實驗過程

### 1. llama.cpp 建置

從 source build（CUDA 12.0），`libggml-cuda.so` 確認存在。
位置：`[[LINUX_PATH_07dc9c]] [[PRIVATE_IP_c0a342]]）

### 2. Mixtral-8x7B — 嘗試失敗（格式不相容）

下載 `Mixtral-8x7B-Instruct-v0.1-Q4_K_M.gguf`（TheBloke / MaziyarPanahi），啟動報錯：

```
missing tensor 'blk.0.ffn_down_exps.weight'
```

**根因：** 2023 年的舊 GGUF 格式，每個 expert 是獨立 tensor（`blk.X.ffn_down.X.weight`）。
新版 llama.cpp（b9781+）期待合併格式（`blk.X.ffn_down_exps.weight`）。HuggingFace 上多數 Mixtral GGUF 都是舊格式，不相容。

**解法：** 改用 `ollama pull mixtral:8x7b-instruct-v0.1-q4_K_M`（Ollama 自己的 registry 有相容版本），在 Ollama port 8000 成功跑起來（10.6GB VRAM）。

### 3. Phi-3.5-MoE-instruct Q4_K_M — 成功啟動

**模型：** Microsoft Phi-3.5-MoE-instruct，42B 參數 / 6.6B active（MoE），Q4_K_M GGUF，24GB。
來源：bartowski HuggingFace repo（`aria2c` 16 連線下載）。

**OOM 踩坑過程：**

| 嘗試 | 設定 | 結果 |
|------|------|------|
| 第一次 | `-ngl 99` | OOM（全塞 GPU，24GB > 12GB） |
| 第二次 | `-ngl 16, n_parallel=4（auto）` | compute buffer OOM（模型勉強塞進去，但 4 slot KV cache 佔爆） |
| 第三次 | `-ngl 16, -np 1` | 仍 OOM（16 層 ~12GB + compute buffers 無空間） |
| **成功** | `-ngl 12, -np 1, -c 4096` | VRAM 9.3GB，啟動成功 |

**啟動參數：**
```bash
~/llama.cpp/build/bin/llama-server \
  -m ~/models/Phi-3.5-MoE-instruct-Q4_K_M.gguf \
  -ngl 12 -c 4096 -np 1 \
  --port 8001 --host 0.0.0.0
```

Port 8001，內網直接可打（`http://[[PRIVATE_IP_c0a342]]:8001/v1`）。
Script 留存：`~/llamaserver.sh`（on [[PRIVATE_IP_c0a342]]）

---

## 速度測試結果

用 Argus 等級的資安分析 prompt（中文，200 tokens output）測試：

| 模型 | 速度 | VRAM |
|------|------|------|
| gemma4:26b（Ollama） | **~35 tok/s** | ~8GB |
| Phi-3.5-MoE（llama-server, -ngl 12） | **~11 tok/s** | 9.3GB |

Phi-3.5-MoE 慢 3 倍的原因：20 層在 CPU RAM，每個 token 生成都有 CPU↔GPU 資料搬移開銷。這是 hybrid offloading 的本質限制，不是設定問題。

---

## 關鍵結論

### 品質 vs 速度 trade-off
Phi-3.5-MoE 回答品質確實比 gemma4 好（更強的推理能力），但 Argus 使用 RTX 做 **streaming final answer**，用戶體感速度是 35 tok/s vs 11 tok/s，差距明顯。

### 現有架構已是正確分工
```
DGX 88B ([[PRIVATE_IP_d54cc2]])  → 重度推理 / tool calling（品質優先，非串流）
RTX gemma4 ([[PRIVATE_IP_c0a342]]) → streaming 最終回答（速度優先）
```

Phi-3.5-MoE 夾在中間：比 gemma4 強、比 88B 弱、速度輸 gemma4。**沒有填到真正的空缺。**

### 真正的問題不是「換更好的 LLM」
想提升 AI 能力，關鍵是 **LLM + 工具整合**（ML/DL pipeline、nginx-shield scoring、ES 查詢），而不是增加更多 LLM 選項讓人選。這個整合已在 Argus/infra-ai 的 tool calling 架構裡。

---

## 其他踩坑記錄

- **wget 在 SSH 斷線後被 kill** → 改用 `tmux + aria2c`（下載存活）
- **Ollama CLI 在 edgexpert 無法連線** → 需 prefix `OLLAMA_HOST=localhost:8000`
- **Ollama KEEP_ALIVE=-1 + 兩個 model server 同時跑** → VRAM OOM 風險，不能讓 llama-server 和 Ollama 同時 loaded
- **llama-server 沒有 `--system-prompt` flag** → system prompt 需在每個 request 的 messages 裡帶

---

## 現狀

- **gemma4 on Ollama port 8000** — 維持，Argus streaming backend
- **Mixtral on Ollama port 8000** — 可用（ollama pull 版本，格式相容）
- **Phi-3.5-MoE GGUF** — 留在 `~/models/`（on [[PRIVATE_IP_c0a342]]），llama-server 已關閉
- **llamaserver.sh** — 留存，未來需要時可重啟

---

## 若未來想重試

適合 Phi-3.5-MoE 的場景：**非串流的批次分析**（不在意速度）。
這時候用 DGX 88B 本來就更好，所以這個場景在現有架構下也不成立。

真正值得重試的條件：拿到 VRAM > 24GB 的機器（能全 GPU），屆時 Phi-3.5-MoE 速度才能展現 MoE 的優勢。
