# gpt-oss 20B + vLLM + GGUF：5 層問題的除錯記錄

> 結論先說：4 個問題修好了，第 5 個 MoE expert weight mapping 卡住，暫時回退 Phi-4。記錄留著，下次繼續。

---

## 動機

Phi-4 14B 指令遵從能力不足（草稿複製原信內容）。gpt-oss 20B 是 OpenAI 開源的 MoE 模型，曾在 Ollama 跑起來且速度快（MoE sparse activation，每次 inference 只用 1/8 參數）。vLLM 0.20.0 有 `GptOssForCausalLM` 原生支援 + 專用 Triton MoE kernel，理論上可以跑。

## 模型資訊

- HF：`openai/gpt-oss-20b`（not gated，tagged `vllm`）
- 架構：MoE 32 experts / 4 active
- GGUF Q4：~13.79GB（從 Ollama blob 轉過來）
- 全精度 safetensors：~43GB，16GB VRAM 塞不下，只能走 GGUF

---

## 解決過程：5 層問題，4 個已修

### ✅ 問題 1：`gguf.__version__` 返回 `'N/A'`

**現象**：transformers 的 `is_gguf_available()` 拿到 `'N/A'` 後 `Version('N/A')` 直接炸

**根本原因**：`gguf` 套件沒有 `__version__` 屬性

**修法**：

```bash
echo '__version__ = "0.18.0"' >> $(python3 -c "import gguf; import os; print(os.path.dirname(gguf.__file__))")/__init__.py
```

---

### ✅ 問題 2：GGUF 架構 `gptoss` 不被 transformers 認識

**現象**：transformers 只檢查 `gpt_oss` 和 `gpt-oss`，但 GGUF 檔案內的 architecture 是 `gptoss`（無底線/連字號）

**修法**：

找到 transformers 的 `modeling_gguf_pytorch_utils.py`，在架構判斷處加入第三種寫法：

```python
# 原本
elif "gpt_oss" in architecture or "gpt-oss" in architecture:

# 修改後
elif "gpt_oss" in architecture or "gpt-oss" in architecture or "gptoss" in architecture:
```

---

### ✅ 問題 3：dtype bfloat16 不支援 GGUF

**現象**：vLLM 用 bfloat16 載入 GGUF 報錯

**根本原因**：vLLM GGUF 量化只支援 float16 / float32，不支援 bfloat16

**修法**：啟動參數加 `--dtype float16`

---

### ✅ 問題 4：model_type `gpt_oss` vs `gpt-oss` 比對失敗

**現象**：transformers 正規化後 model_type = `gpt_oss`，但 `gguf.MODEL_ARCH_NAMES` 是 `gpt-oss`，完全比對失敗

**修法**：

找到 vLLM 的 `vllm/model_executor/model_loader/gguf_loader.py`，在比對處加入 hyphen/underscore 正規化：

```python
# 原本
if value == model_type:

# 修改後
if value == model_type or value.replace("-", "_") == model_type or value == model_type.replace("_", "-"):
```

---

### ❌ 問題 5：MoE expert weight 名稱對應（未解決）

**錯誤**：

```
RuntimeError: Failed to map GGUF parameters (72):
['model.layers.0.mlp.experts.gate_up_proj',
 'model.layers.0.mlp.experts.gate_up_proj_bias',
 'model.layers.0.mlp.experts.down_proj_bias', ...]
```

**根本原因**：

- GGUF 存的是 **merged expert weights**（`mlp.experts.gate_up_proj`，所有 expert 合併成一個 tensor）
- vLLM HF model 期待 **split weights**（`mlp.experts.0.gate_proj.weight`、`mlp.experts.1.gate_proj.weight`... 每個 expert 分開）
- `gguf_loader.py` 裡沒有 gpt_oss 的 MoE expert weight 對應表

**下一步**：在 `gguf_loader.py` 的 `_get_gguf_weights_map()` 加入 gpt_oss 的 sideload_params，做法類似 deepseek2 的實作。

---

## 已修改的位置（給自己備查）

| 檔案 | 修改內容 |
|------|----------|
| `gguf/__init__.py` | 加 `__version__ = "0.18.0"` |
| `transformers/modeling_gguf_pytorch_utils.py` | 架構比對加入 `gptoss`（無連字號版） |
| `vllm/model_executor/model_loader/gguf_loader.py` | model_type 比對加入 hyphen/underscore 正規化 |

---

## 啟動參數（問題 5 解決後使用）

```
--model /path/to/gpt-oss-20b/model.gguf
--tokenizer /path/to/gpt-oss-20b
--load-format gguf
--trust-remote-code
--dtype float16
--port 11434
--host 0.0.0.0
--gpu-memory-utilization 0.90
--max-model-len 8192
```

---

## 現況

回退至 Phi-4 14B AWQ 繼續穩定運行。GGUF + tokenizer 保留在本機，四個 patch 已套用，下次繼續從問題 5 的 MoE weight mapping 開始。

---

*vLLM 對 GGUF 的支援還不夠完整，gpt-oss 這種 MoE 架構需要額外的 weight 對應表，官方目前沒有。有人踩過同樣的坑歡迎交流。*
