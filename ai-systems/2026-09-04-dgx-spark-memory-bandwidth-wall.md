# 4 台 DGX Spark 想服務 RD？記憶體頻寬這關過不了

> 寫於 2026-09-04，規劃階段的實測記錄，留著回測用。

---

想找一個便宜的地端方案，讓 30 個以上的 RD 用自架 AI 做 agentic 開發。桌上型的 DGX Spark（GB10，128GB 統一記憶體）一台不到伺服器等級的價格，4 台串起來聽起來剛好。

我一開始想的是容量夠不夠、算力夠不夠。都想錯了。

---

**decode 是記憶體頻寬綁定的。** 每生一個 token，要把當下用到的模型參數從記憶體整個讀一遍。所以每人拿到的 token/s，上限就是「有效頻寬 ÷ 每 token 要讀的 bytes」。

DGX Spark 用的是 LPDDR5X，理論 273 GB/s，實測大概 200–240。對照組：RTX Pro 6000 的 GDDR7 約 1,800，H200 的 HBM3e 約 4,800。差 7 到 17 倍。

而且 4 台不能合起來當一台用——單元間只有 200GbE（25 GB/s），是本機頻寬的十分之一。串起來跑一顆大模型會被互連卡死。實際能做的只有「4 台各跑一個獨立 replica」，等於 4 座 220 GB/s 的孤島。

---

紙上算下來是「勉強到不夠」。與其猜，不如量。

寫了一支並發壓測腳本，打我們已經有的那台 edgexpert（同款 GB10），掃 1 到 12 條並發、餵約 20k token 的 coding context、每題生成 400 token。

結果比估算還硬，而且兩顆模型撞牆的方式不一樣：

| 並發 | Qwen3.8:27b（full attention） | gemma4:26b（sliding window） |
|---|---|---|
| 1 | 25 tok/s | 48 tok/s |
| 2 | 25 | **14** |
| 6 | 25（首 token 等 82 秒） | 13.5 |
| 12 | 25（首 token 等 **179 秒**） | 13.7 |

**Qwen3.8：完全序列化。** full-attention 的 KV cache 在 20k context 太重，Ollama 實際只開 1 個 slot。每人固定 25 tok/s，但請求一個一個做——加人只是排更長的隊。整台 decode 總量卡在 24 tok/s。

**gemma4：會批次，但每人腰斬。** sliding-window attention 讓 KV 夠小、批次有效，整台總量能爬到約 80 tok/s。但 220 GB/s 就這麼多，第二個人一進來，每人立刻從 48 掉到 14，之後一路平在那。

DGX Spark 沒有中間帶：要嘛一個人全速，要嘛一群人各 14 tok/s。

---

4 台的實際容量：

- gemma 級模型：4 台聚合約 320 tok/s → 10 個人各 30 tok/s，或 30 個人各 11 tok/s。
- 要 30 個人同時各 30 tok/s（約 900 tok/s 聚合）→ 得 11–12 台 gemma 級 Spark，或一張 RTX Pro 6000。

「4 台 DGX Spark 服務 30 人 RD」這個想法，到這裡結束。

---

留著回測的幾點：

1. **LPDDR5X 的頻寬天花板是物理的。** 量化、批次、串接都繞不過去。買之前先知道這件事。
2. **attention 架構在頻寬受限的硬體上是決定性的。** full-attention 的 27B 在 GB10 上直接不能多人用；同樣大小的 sliding-window 模型還能撐。挑模型不能只看參數量。
3. **單串流數字會騙人。** gemma4 單人 48 tok/s 看起來很好，兩個人就崩一半。要服務多人就一定要測並發。
4. **Ollama 的 `OLLAMA_NUM_PARALLEL` 是硬限制。** agentic 一輪好幾次 tool call，首 token 等三分鐘不能用。多人 serving 要換 vLLM 的連續批次。
5. **DGX Spark 的定位是「單一團隊的快速助手」**（就是 edgexpert 現在在做的事）、dev/CI、邊緣節點——不是幾十人共用的 serving fabric。

便宜的硬體省的是採購單上的數字。頻寬省不了。

---

還沒做的：把 `Qwen3-Coder-30B-A3B`（3B 活躍 + 強 GQA）拉上 edgexpert 用同一支腳本重測，看小 active 的 MoE 能不能把每人速率推回 30；同時測 vLLM 連續批次能不能解掉 Ollama 那個排隊問題。預期會比 gemma4 好，但不會翻盤「4 台不夠」的結論。
