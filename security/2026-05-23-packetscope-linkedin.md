# 一個 LinkedIn 訊息，當天晚上建了一個 repo

> 寫於 2026-05-23，留著回測用。

---

今天有人在 LinkedIn 找我聊，說對我貼的 NGINX AI 防禦架構有興趣。

對話很快進入技術層面——FortiGate 的加密流量可見度、TLS fingerprint、packet-level 分析、怎麼從 pcap 裡找出被審查的流量特徵。

對方在做守方工具，從客戶端視角觀察網路路徑：RST timing、TTL pattern、silent drop——目的是讓防禦方看懂自己的流量在哪裡被干擾、被 RST、被靜默丟棄。

聊著聊著我建議他用 Isolation Forest 做噪音過濾、Random Forest 做分類、Change Point Detection 抓行為漂移。這個組合對 packet-level 分析比 LSTM 或 HMM 更實際。

當天晚上我把這個架構做成一個概念 repo 丟給他：[PacketScope](https://github.com/kerr20801/PacketScope)。

---

PacketScope 是守方工具。

把 tcpdump 輸出貼進去，10 秒內看到：C2 beaconing、port scan、data exfil、DNS tunnel——是 SOC 分析師在沒有 SIEM 時用的東西，不是攻擊工具。

在資安這個領域，工具的定位要說清楚。守方工具不需要因為「也許有人拿去做壞事」就說自己是雙用途——螺絲起子也能當武器，但它就是螺絲起子。

---

技術可以分享，定位要說清楚。

這個原則在 2026 年不會變，五年後也不會。
