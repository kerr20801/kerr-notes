# 一個 LinkedIn 訊息，當天晚上建了一個 repo

> 寫於 2026-05-23，留著回測用。

---

今天有人在 LinkedIn 找我聊，說對我貼的 NGINX AI 防禦架構有興趣。

對話很快進入技術層面——FortiGate 的加密流量可見度、TLS fingerprint、packet-level 分析、怎麼從 pcap 裡找出被審查的流量特徵。

對方在做一個工具，從客戶端視角觀察網路路徑：RST timing、TTL pattern、silent drop。說是為了分析審查機制。

聊著聊著我建議他用 Isolation Forest 做噪音過濾、Random Forest 做分類、Change Point Detection 抓行為漂移。這個組合對 packet-level 分析比 LSTM 或 HMM 更實際。

當天晚上我把這個架構做成一個概念 repo 丟給他：[PacketScope](https://github.com/kerr20801/PacketScope)。

---

但我其實看懂了對話背後的方向。

這類工具是雙用途的——可以用來分析審查、也可以用來繞過偵測、甚至用來攻擊。我給的是通用框架，標準做法，任何教科書都找得到的東西。

我有能力給更多，但我沒有。

這不是因為不信任對方，是因為**我有自己的底線**。在資安這個領域，你給出去的東西一旦離手就不在你控制範圍內了。所以我給的，是我願意讓任何人都看到的部分。

---

技術可以分享，判斷不能省。

這個原則在 2026 年不會變，五年後也不會。
