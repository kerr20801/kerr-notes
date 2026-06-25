# 多個 AI 平台換模型，為什麼不用 LiteLLM

> 寫於 2026-06-25

---

我們內部跑了四個 AI 平台：infra-ai（基礎設施執行 AI）、Argus（資安監控）、CTI AI（威脅情資研究）、Red Team AI（滲透測試）。

每個平台都有自己的 LLM backend，而且每個平台都 hardcode 了 model name：

```python
# 這四個 app 各自寫了類似的東西
DGX_MODEL = "nvidia/gpt-oss-puzzle-88B"
RTX_MODEL = "gemma4"
EDGE_MODEL = "gpt-oss-fast:latest"
```

換模型的時候，四個地方都要改。如果有 typo、有人忘記改、staging 和 prod 不一致，就出問題。

---

## 直覺解法：LiteLLM

LiteLLM 是個 proxy，你定義邏輯名稱，它對應到真實 backend：

```yaml
model_list:
  - model_name: t5/reasoning
    litellm_params:
      model: nvidia/gpt-oss-puzzle-88B
      api_base: http://10.254.1.17:8000/v1
```

App 只需要打 `http://localhost:4000/v1`，model name 用 `t5/reasoning`。換模型改 yaml 就好，四個 app 不動。

聽起來很完美。但有個問題我一開始沒注意到。

---

## LiteLLM 擋掉了什麼

我們有個需求：**同時送給多個 cloud 模型，比對答案**。

現在只有一個 cloud（Gemma4 Cloud），但以後可能有 cloud2、cloud3。Red Team AI 的「戰情室」設計就是這樣——攻擊者、覆核者、策略官三個模型同時跑，接力辯論。

LiteLLM 是 1 對 1 路由。`t5/cloud1` → 一台機器。如果要比較，你得 app 自己呼叫 `t5/cloud1`、`t5/cloud2`……那 LiteLLM 就只是個 proxy，不是你真正需要的東西。

更重要的問題：現在加一層 gateway，等於加一個需要維護、需要監控、會故障的服務。為了解決 model name 的問題，不值得。

---

## 實際做法：server alias

最輕的解法：讓每台 LLM server 直接認識統一的名稱。App 永遠用固定名稱，server 那邊決定實際是哪個模型。

定義三個角色：

| 別名 | 角色 | 現在指向 |
|------|------|---------|
| `t5/reasoning` | 重推理 | DGX `nvidia/gpt-oss-puzzle-88B` |
| `t5/fast` | 快速生成 | edgexpert `gpt-oss-fast:latest` |
| `t5/stream` | 串流回答 | RTX `gemma4:latest` |

**vLLM（DGX）**：加 `--served-model-name` 啟動參數，可以同時接受多個名稱：

```json
"always_serve_args": [
    "--served-model-name", "nvidia/gpt-oss-puzzle-88B",
    "--served-model-name", "t5/reasoning"
]
```

換模型時，`--model` 改新的，`--served-model-name t5/reasoning` 保留不動。

**Ollama（edgexpert / RTX）**：用 `ollama create` 建 alias：

```bash
# 取出 modelfile，FROM 改成 tag（不能用 blob 路徑，permission denied）
OLLAMA_HOST=localhost:8000 ollama show gpt-oss-fast:latest --modelfile > /tmp/m.modelfile
sed -i 's|FROM /usr/share/ollama/.*|FROM gpt-oss-fast:latest|' /tmp/m.modelfile
OLLAMA_HOST=localhost:8000 ollama create t5/fast -f /tmp/m.modelfile
```

換模型時，`FROM` 改新 model，`ollama create t5/fast` 重跑。

---

## Cloud 模型的特殊問題

`llm-gemma4.teamt5.work`（外部 API）只認識 `google/gemma-4-26B-A4B-it`，我們沒辦法在那邊建 alias。

解法：分開兩個概念。

```python
GEMMA4_CLOUD_MODEL     = "t5/cloud1"            # 邏輯別名，app 內部選擇用
GEMMA4_CLOUD_API_MODEL = "google/gemma-4-26B-A4B-it"  # 實際送給 API 的名稱

# 發 request 時
payload["model"] = GEMMA4_CLOUD_API_MODEL
```

Config 格式給其他人參考用的：

```json
{
  "t5/cloud1": {
    "name": "Gemma 4 31B (TeamT5 TPU)",
    "api_model": "google/gemma-4-26B-A4B-it",
    "url": "https://llm-gemma4.teamt5.work/v1"
  }
}
```

未來加 `t5/cloud2`：在 config 多一條，比較迴圈自動涵蓋。這才是設計上真正該有的彈性。

---

## 結果

四個 app 各改了一次，之後永遠不用動：

```python
# 改前
DGX_MODEL = "nvidia/gpt-oss-puzzle-88B"

# 改後
DGX_MODEL = "t5/reasoning"
```

DGX 重啟、edgexpert 的 Ollama 新增兩個 alias、RTX 一個，全部完成。

順便修掉一個藏了一段時間的 bug：Argus 的 `OLLAMA_URL` 還指著 RTX 的舊 port 11434（Ollama 已換成 8000），一直沒人發現，因為那條 code path 不常走。

---

## 這個設計的核心假設

**「換模型」和「比較模型」是不同問題。**

前者只需要 alias（server 端改）。後者是 orchestration 邏輯，屬於 app 層，不該推給 proxy 解決。

LiteLLM 在前者很好，但如果你的系統往後會做多模型協作，加它只是在中間加一道牆，而且牆上寫的是「這裡只能 1 對 1」。

---

最輕的解，往往是讓基礎設施自己吸收複雜度，不是在上面再疊一層。
