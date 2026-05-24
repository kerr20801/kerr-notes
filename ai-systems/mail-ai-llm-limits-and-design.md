# Mail AI：小模型的規則上限，以及繞過它的設計決策

## 系統是什麼

Outlook 信件。自動讀信、判斷要不要回、產草稿，看完按寄出。
FastAPI server + Ollama，Windows 客戶端，完全在內網跑。

---

## 根本問題：14B 模型同時遵守 15 條規則的能力有上限

Phi-4 14B 時期，prompt 加了約 15 條規則（語言、稱呼、格式、長度、禁止事項...）。症狀：

- 英文來信仍回英文（rule: 永遠用繁體中文）
- 稱呼叫錯人（rule: 從簽名萃取 greeting name）
- 重複同一句 15 次（rule: 不得重複）
- 複製原信內容（模型容量不足，遵守規則已耗盡注意力）

加更多規則，邊際效益遞減。這不是 prompt 工程問題，是模型容量問題。

**結論**：小模型能同時遵守的規則數有上限，超過之後規則互相干擾，加愈多愈亂。

換更大的模型（gpt-oss-20B MoE）之後，大多數規則開始生效，繁體中文、稱呼、格式都穩了。
但 20B MoE 14GB VRAM，已是這台機器的極限，再大就裝不下。

---

## 設計決策一：Model Alias，解耦程式碼和模型

每次換模型，所有呼叫 LLM 的程式碼都要改 model name。Mail AI、C.K、其他工具各自有 hardcode。

解法：在 Ollama 建一個 alias，永遠叫同一個名字：

```bash
# Modelfile
FROM /path/to/actual-model.gguf

# 建立 alias
ollama create "devstral-small-2:24b" -f Modelfile
```

所有程式碼寫死 `devstral-small-2:24b`，換模型只需重建 alias，code 不動。

這個做法的前提是 Ollama，vLLM 沒有這個機制（vLLM 要改 `--model` 參數重啟）。這也是最後選回 Ollama 的原因之一。

---

## 設計決策二：action_classifier 在 LLM 之前過濾

不是每封信都需要回。LLM 最貴的不是 token，是「產了沒用的草稿」浪費 Jessie 的判斷時間。

在送進 LLM 之前，rule-based classifier 先判斷要不要處理：

```python
# 信是寫給別人的，user 只是 CC
if greeting_name not in {'user', 'user xxx', '使用者中文名'}:
    return False, 'low'

# Email @mention 別人
if re.match(r"hi\s+@['\"]?\w.*@\w", first_line, re.I):
    return False, 'low'

# 內部轉發通知
if '【XXX internal】' in subject:
    return False, 'low'
```

規則寫死，快，不耗 VRAM。LLM 只處理真的要回的信。

這個分層很關鍵：**確定性規則做過濾，LLM 做生成**。不要把過濾邏輯也丟給 LLM，它會偶爾判斷錯，而且每封信都多花一次推理。

---

## 設計決策三：垃圾偵測 + fallback

LLM 有時輸出空字串、或單字片段組成的亂碼（`我 會 在 D t v 後 為 此 項目 產生 綠色 檔案`）。
這種輸出送進去 Outlook 草稿，比沒有草稿更糟。

偵測邏輯：

```python
def _is_garbage(text):
    words = text.split()
    single = sum(1 for w in words if len(w) == 1)
    # 單字元佔 40% 以上 → 垃圾
    if len(words) > 5 and single / len(words) > 0.4:
        return True
    # 3+ 連續單字元
    if re.search(r'(?:^|\s)\S(?:\s+\S){3,}(?:\s|$)', text):
        return True
    return False
```

偵測到垃圾就 fallback 到固定模板（`Hi {name}, 收到，我確認後回覆。`），至少不丟一個爛草稿給 user。

---

## Ollama vs vLLM

一開始用 vLLM 是為了效能。後來換回 Ollama：

| | vLLM | Ollama |
|--|------|--------|
| GGUF 支援 | ❌ 要轉格式 | ✅ 直接跑 |
| MoE 模型 | loader 有 bug | ✅ 穩定 |
| Model alias | ❌ 無 | ✅ Modelfile |
| 多模型切換 | 要重啟 | `ollama list` 即切 |
| 設定複雜度 | 高 | 低 |

在內網單機、一次只服務一個 user 的場景，Ollama 的吞吐量完全夠。vLLM 的優勢（高並發、continuous batching）在這個規模根本用不到。

---

## 目前還沒解的問題

- 簡體來信偶爾仍回簡體（rule 14 壓不住）
- 複雜多人信（3+ 人 CC）品質不穩定
- 20B 已是這台機器的極限，更好的品質需要換硬體或用 API
