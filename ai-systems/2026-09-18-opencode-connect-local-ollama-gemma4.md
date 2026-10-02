# OpenCode 接地端 Gemma4：設定記錄，順便測出比硬接 Claude Code 快 5 倍

> 寫於 2026-09-18

---

起因是想拿自己家裡跑的 gemma4:26b（Ollama）接不同的終端機編碼 agent 玩玩看。先試了看起來最直覺的路——直接把 Claude Code 接過去——結果繞了一圈才發現這條路不對，換成 OpenCode 才是真的順。整個過程連設定帶實測記下來。

---

## 一、先試 Claude Code 硬接，能動但代價不小

Ollama 從 v0.14（2026/1）開始原生支援 Anthropic Messages API 相容層，設定很簡單：

```bash
export ANTHROPIC_BASE_URL=http://127.0.0.1:11434
export ANTHROPIC_AUTH_TOKEN=ollama   # 本機不驗證，隨便填
claude --model gemma4-redteam:26b -p "你的問題"
```

跑是跑得動，工具呼叫（讀檔案、跑指令）也真的能用。但兩個問題：

**沒有 prompt caching。** Claude Code 每一輪都要重送一次完整的 system prompt + 工具定義，沒有快取機制，代表每次都整包重算。一個「讀一行檔案 + 答 1+1」這種級別的請求，實測 **93 秒**。

**Claude Code 不認得這顆模型，自己猜了一個錯的 context window。** 它的模型清單裡沒有 `gemma4-redteam:26b` 這個名字，判斷不出實際視窗大小，預設假裝是 **200K**——但這顆模型的 Modelfile 裡設的其實是 **32768**。長對話跑到後面，Claude Code 以為還有很多空間，實際上早就超過模型真正能吃的量，會在某個時間點無預警壞掉，不會事先示警。

這是因為 Claude Code 骨子裡是為 Anthropic 自家模型設計的，本地模型相容是後補上去的一層殼，套進去能用，但底層假設沒跟著換。

---

## 二、換 OpenCode：原生多供應商，不用套殼

OpenCode 從設計上就是供應商中立，直接打 Ollama 原生的 OpenAI-compatible `/v1` endpoint，不假裝自己是誰。設定寫在 `~/.config/opencode/opencode.json`：

```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "home-ollama": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Home Ollama (gemma4)",
      "options": {
        "baseURL": "http://172.16.32.116:11434/v1",
        "apiKey": "ollama"
      },
      "models": {
        "gemma4-redteam:26b": {
          "name": "gemma4-redteam 32K",
          "options": {
            "reasoning_effort": "none"
          }
        }
      }
    }
  }
}
```

`baseURL` 指到跑 Ollama 那台機器的 `/v1`，`models` 底下填實際的模型 tag。存檔後 `opencode models` 應該會列出 `home-ollama/gemma4-redteam:26b`，代表接上了。

沒裝過 OpenCode 的話，Ollama 官方有包好的一行指令：`ollama launch opencode`。

兩個設定值不能省：

**`apiKey` 隨便填也要填。** Ollama 完全不驗證這個值，但 `@ai-sdk/openai-compatible` 這個 provider 類型的介面本來就預期要有一個值。不填的話，在 OpenCode 互動介面裡選這個 provider 時會卡在要求輸入 API key 的畫面——填任何字串都能過，只是不能空著。

**gemma4 是會思考的模型，不關思考會在回應裡吐一堆推理過程。** 這是我們家其他系統早就踩過的坑，這次在 OpenCode 上又中一次。修法是在對應模型底下加 `options.reasoning_effort: "none"`——`@ai-sdk/openai-compatible` 會把這個欄位原封不動塞進送給 Ollama 的請求 body，跟直接用 curl 打 `/v1/chat/completions` 時要加的參數一樣。加了之後問「7×8」只會回 `56`，不會回一整段思考。

跑起來：

```bash
opencode run "你的任務" --model home-ollama/gemma4-redteam:26b
```

官方文件有提一句：工具呼叫（tool call）品質不好的話，把 `num_ctx` 拉到 16K~32K 以上會有幫助——這也是為什麼我特地挑了 32K 那顆 alias 測，不是拿一般互動用的 8K 那顆。

---

## 三、實測對比：任務更重，時間卻少了 5 倍

兩邊都是接同一台機器的同一顆模型（`gemma4-redteam:26b`），做的事完全不對等——刻意讓 OpenCode 那邊做更重的任務：

| | Claude Code（Anthropic 相容殼） | OpenCode（原生協定） |
|---|---|---|
| 任務 | 讀一行檔案 + 答 1+1 | 寫一支 `fib.py`（memoization）+ 實際執行 + 驗證結果 |
| 耗時 | 93 秒 | **16.8 秒** |
| 結果 | 正確 | 正確 |

OpenCode 產出的程式碼：

```python
from functools import lru_cache

@lru_cache(None)
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

if __name__ == "__main__":
    result = fibonacci(20)
    print(result)
```

`fibonacci(20) = 6765`，算對了，也真的用了要求的 `functools.lru_cache`。

同一顆模型，換一個講原生協定的殼，時間差了一個量級，做的事還比較多。差距不是模型變強了，是少了一層「假裝自己是別人」的轉譯稅。

---

## 補記（2026-09-19）：升級 OpenCode 版本後，同一個任務變慢了

裝的時候發現 OpenCode 已經**八個月沒更新**（1.1.26，2026/1 裝的），升到最新版 1.18.31 後拿同一個任務重測兩次：52.5 秒、30.2 秒，都比舊版的 16.8 秒慢。

查了一下不是模型的問題——直接繞過 OpenCode 對 Ollama 打同等級的原始請求，`real` 時間 22 秒左右，但本機 CPU 時間（`user`）幾乎是 0，這才是正常的「純網路等待」樣子。新版 OpenCode 兩次測試的 `user` 時間都停在 15~20 秒，代表 client 端自己在本機多算了什麼，跟模型生成速度無關。

具體是哪個功能造成的還沒查，先誠實記下：**這篇的「16.8 秒」是舊版本的數字，目前最新版實測會更慢。** 之後查出原因再回來補。

---

## 收尾

想把地端模型接進終端機編碼 agent，先看那個 agent 是不是原生支援多供應商（OpenCode、部分其他工具是），而不是找一顆專門為某家雲端模型設計、後來才勉強加上相容層的工具去硬套——後者能動，但你會付兩筆稅：沒有 caching 的速度稅，跟工具自己對模型能力猜錯的穩定性稅。

---

*相關：[「本地 AI 太笨」很多時候只是「沒有應用架構」的另一種說法](2026-08-19-llm-is-not-the-application.md)*
