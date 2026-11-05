# ML 不能排第一層：當你的工具在陌生環境盲跑時

## 背景

跟一個工程師聊網路偵測工具。他在做 dpi-probe，從客戶端視角探測網路路徑行為——TCP RST timing、TLS fingerprint、SNI silent drop、TTL 異常。生活在高審查環境，需求很真實。

我從基礎設施內部做 behavioral detection：FortiGate + NGINX log → ELK → ML 時序分析。

兩個人方向剛好對稱，解的是同一道題。

## 他問的問題

> 你的 ML 層怎麼處理漸進式漂移？模型是定期重訓還是事件驅動？

## 我的答案，以及反過來給他的建議

我們的做法：長期 baseline + ELK 時序分析，偵測統計變異。

但反過來跟他說：**你不應該把 ML 放在第一層。**

原因：ML-first 只在你控制「正常」的環境下有效。他的工具在任意網路盲跑——不知道那個 ISP 的 baseline 是什麼，不知道那段路徑的正常 RTT 是多少。在建立 ground truth 之前，模型沒有東西可以學。

```
deterministic evidence chain 先走
  → 每次 probe 結果有明確的觀察（RST? timeout? TLS alert?）
  → 累積足夠不同環境的資料
  → 這時候 ML 才有意義
```

他的工具「盲跑」，我的工具「在已知環境內跑」。這個差異決定了 ML 在 pipeline 裡的位置。

## Phase 2 不只是 engineering hygiene

他的 roadmap Phase 2 在做：穩定 JSON schema、per-signal confidence、把觀察和詮釋分開。

他說沒想到這些其實是在建未來的訓練資料結構。

對。`clean/blocked domain split` 就是隱性 label。`ISP + path + timestamp` 就是環境脈絡。資料結構穩定了，未來的 ML 層不需要重新整理資料，直接接上去。

## 資料冷啟動問題

他之後想做 voluntary 匿名分享：本地跑、本地產生 sanitized report、移除 IP 和精確時間戳、使用者自己決定要不要貢獻。

一個值得記的觀點：**export format 應該早點公開，不用等平台建好。**

schema 穩定、文件清楚，研究者可以開始手動交換資料。dataset 在任何中央系統存在之前就能自己長出來。OONI 有規模但深度不夠，這種工具可以有深度，但使用者天生就會少——這個 trade-off 從一開始就要認清。
