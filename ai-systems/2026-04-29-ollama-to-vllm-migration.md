# Ollama → vLLM 遷移記錄

> 結論先說：vLLM 比 Ollama 快，API 相容 OpenAI 格式，但有幾個踩坑點——Blackwell GPU 必須用 open kernel module、AWQ 要用 awq_marlin 不是 awq、port 維持不變讓上層程式碼零修改。

---

## 背景

原本用 Ollama 跑 24B 量化模型（GGUF Q4_K_M）。今天完整遷移至 vLLM 0.20.0。

遷移動機：
- vLLM 對 AWQ 量化有更好的 kernel 支援（awq_marlin）
- OpenAI-compatible API，整合其他工具更方便
- 推論速度比 Ollama 快

---

## 前置作業：Blackwell GPU Driver 修復

重開機後 `nvidia-smi` 無法偵測 RTX 5060 Ti，dmesg 顯示：

```
NVRM: installed in this system requires use of the NVIDIA open kernel modules.
```

**根本原因**：Blackwell 架構（RTX 50 系列）必須使用 open kernel modules，不能用舊的 closed-source 版本。

**修法：**

```bash
# 複製 DKMS 已建好的 open module 到高優先路徑
sudo mkdir -p /lib/modules/$(uname -r)/updates/dkms/
sudo cp /var/lib/dkms/nvidia/580.142/$(uname -r)/x86_64/module/nvidia*.ko \
  /lib/modules/$(uname -r)/updates/dkms/
sudo depmod -a $(uname -r)
sudo rmmod nvidia_drm nvidia_modeset nvidia_uvm nvidia
sudo modprobe nvidia

# 永久化
sudo apt install linux-modules-nvidia-580-open-$(uname -r) linux-modules-nvidia-580-open-generic
```

往後若重開機後 GPU 掛掉，確認 `/lib/modules/$(uname -r)/updates/dkms/nvidia.ko` 存在，license 應為 `Dual MIT/GPL`。

---

## 模型選型

| 模型 | 大小 | VRAM 需求 | 結果 |
|------|------|-----------|------|
| devstral-24B AWQ | 18GB | 15.1GB（vision encoder fp16 未量化） | ❌ OOM，無 KV cache 空間 |
| Gemma 4 E4B | 9.6GB (BF16) | ~10GB | 可用但只有 4B，能力不足 |
| Phi-4 14B AWQ Marlin | 8.6GB | ~14GB | ✅ 採用 |

**選 Phi-4 14B AWQ Marlin 的原因：**
- 8.6GB 模型大小，VRAM 剩下足夠的 KV cache 空間
- AWQ Marlin kernel 比一般 AWQ 快
- 14B 參數對多數任務夠用

模型：`curiousmind147/microsoft-phi-4-AWQ-4bit-GEMM`

---

## vLLM 安裝與設定

```bash
pip install vllm
```

**systemd service（`/etc/systemd/system/vllm.service`）關鍵參數：**

```
--model /path/to/phi4-awq-4bit
--quantization awq_marlin        ← 不是 awq，用 awq_marlin 速度更快
--dtype float16                  ← AWQ Marlin 不支援 bfloat16
--port 11434                     ← 維持 Ollama 原本的 port，上層程式碼不用改
--host 0.0.0.0                   ← 允許其他機器連入
--gpu-memory-utilization 0.92
--max-model-len 16384
--served-model-name <原本的模型名稱>  ← 維持舊名稱，程式碼 model 參數不用改
```

```bash
sudo systemctl enable vllm
sudo systemctl start vllm
```

執行後 VRAM 使用：~14.3GB，剩 ~2GB 給 KV cache。

**讓上層零修改的兩個關鍵：**
1. `--port 11434`：URL 不變
2. `--served-model-name`：model 名稱不變

---

## API 格式差異（Ollama vs vLLM）

從 Ollama 遷移到 vLLM，API 格式要更新：

| 項目 | Ollama（舊） | vLLM（新） |
|------|-------------|-----------|
| 端點 | `/api/generate` | `/v1/completions` |
| 聊天端點 | `/api/chat` | `/v1/chat/completions` |
| 參數 | `options.num_predict` | `max_tokens` |
| 參數 | `options.temperature` | `temperature`（頂層） |
| 回應 | `data["response"]` | `data["choices"][0]["text"]` |
| 聊天回應 | `data["message"]["content"]` | `data["choices"][0]["message"]["content"]` |
| JSON mode | `"format": "json"` | `"response_format": {"type": "json_object"}` |
| 模型列表 | `/api/tags` | `/v1/models` |
| Health check | `/api/tags` | `/v1/models` |

---

## Ollama 清除

```bash
sudo systemctl disable ollama
sudo systemctl stop ollama

# 舊 GGUF 模型（如已不需要）
# sudo rm -rf /usr/share/ollama/.ollama/models/blobs/

# 舊 AWQ 嘗試檔案（如已不需要）
# sudo rm -rf /path/to/old-model/
```

---

## 注意事項

- `awq_marlin` 必須搭配 `float16`，不能用 `bfloat16`
- 多個服務共用同一個 vLLM instance 時，KV cache 共享，並發請求會排隊但不會 OOM
- Blackwell（RTX 50 系列）GPU 一定要 open kernel module，這不是選項
