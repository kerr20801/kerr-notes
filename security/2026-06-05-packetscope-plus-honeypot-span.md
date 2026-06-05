# PacketScope Plus：為 Honeypot + SPAN Port 設計的 ML 流量分析

> 結論先說：規則制偵測在 Honeypot 場景有一個根本問題——攻擊者的行為沒有固定模式，規則永遠在追。PacketScope Plus 改用無監督 ML，不依賴規則，讓資料自己說哪個 Flow 奇怪。

---

## 為什麼 Honeypot 需要 ML 分析

Honeypot 的流量跟一般內網不同：連進來的沒有一個是「正常使用者」，全部都是可疑的。

這個特性讓規則制偵測效果打折——規則是用來區分正常和異常的，但 Honeypot 裡沒有正常可以當基線。更大的問題是，進階攻擊者的流量特徵刻意設計成看起來正常：心跳間隔加了抖動、封包大小模仿瀏覽器、Port 用 443 而不是 4444。

無監督 ML 的切入點不同：不需要告訴它什麼是惡意，只需要找「跟其他 Flow 不一樣的」。

---

## 設計場景

```
Internet
    ↓
Switch WAN Port
    ├── 正常流量 → Firewall → 內網
    └── SPAN Mirror → 分析機
                        ↓
                   tcpdump 持續抓包
                        ↓
                   PacketScope Plus
                        ↓
                   風險排行 + 告警
```

SPAN port 把 WAN 流量完整複製一份，不影響正常封包路徑。分析機跑 tcpdump 加 PacketScope Plus，看到的是所有進出 WAN 的流量，不只是 Honeypot 接到的部分。

這個架構的優點：攻擊者即使沒打到 Honeypot，只要出現在 WAN 流量裡就會被分析到。

---

## 雙軌 ML 架構

### 軌道 A：Flow 異常偵測（ECOD + Isolation Forest）

把封包聚合成 Flow（5-tuple + 30 秒時間窗），提取 20+ 個特徵：

- **封包統計**：byte_rate、avg_pkt_size、size_entropy
- **TCP Flag 比例**：syn_ratio、rst_ratio（高 RST = 掃描特徵）
- **時序**：inter_arrival_cv（低 CV = 固定間隔 = Beaconing 嫌疑）
- **Half-flow**：SYN 沒有對應 SYN-ACK（掃描行為）

ECOD 不需要調參，直接對特徵分布建模，找尾端的離群值。Isolation Forest 用隨機分割，孤立容易被隔離的點。兩者分數加權平均，ECOD 主導（0.6），IF 輔助（0.4）。

### 軌道 B：Graph 節點分析（PageRank）

把所有 Flow 建成有向圖：節點是 IP，邊是流量，權重是封包數。

計算每個 IP 的 PageRank——高 PageRank 代表大量其他節點通過此 IP 中轉，這是 **Pivot / 跳板節點**的特徵。

橫向移動在 Graph 上特別明顯：攻擊者進入內網後，會從一台機器連到多台，那台「轉發點」的 out-degree 和 PageRank 都會升高，純規則很難發現，Graph 直接浮出來。

### 融合：LightGBM 軟標籤排序

兩軌分數加上原始特徵，丟進 LightGBM 回歸。

沒有標注資料怎麼訓練？用規則產生「軟標籤」當 proxy target：

```python
soft_label = (
    track_a * 0.4 +
    track_b_src * 0.3 +
    is_high_risk_port * 0.2 +
    syn_ratio * 0.1
)
```

LightGBM 學習各特徵的非線性組合，輸出比單純加權更細緻的風險分數。每次跑都重訓（資料量小，幾秒內完成），不需要維護模型版本。

---

## 實際跑起來的樣子

```
============================================================
  PacketScope Phase 3 — 異常偵測
  Flow 數量：1,247，特徵維度：20
============================================================
[track_a] ECOD+IF 完成，高分 Flow（>0.7）: 43 / 1247
[track_b] Graph 完成，高中心度節點（>0.5）: 7 / 312 個 IP
[fusion]  LightGBM Top 特徵: track_a=312 syn_ratio=89 src_out_deg=67...

============================================================
  風險排行 Top 10
============================================================
  🔴 [CRITICAL] 0.934  203.x.x.x → 192.168.1.10:4444  TCP  pkts=847  A=0.91 B=0.78  ⚠ SUSP_PORT
  🔴 [CRITICAL] 0.891  45.x.x.x  → 192.168.1.0/24:22  TCP  pkts=523  A=0.88 B=0.82
  🟠 [HIGH    ] 0.743  198.x.x.x → 192.168.1.10:443   TCP  pkts=214  A=0.71 B=0.31
  ...
```

---

## 跟 HTML 版 PacketScope 的定位差異

| | index.html | src/ (Plus) |
|---|---|---|
| 安裝 | 零依賴，開瀏覽器 | pip install |
| 使用方式 | 貼上封包看結果 | CLI + 檔案路徑 |
| 偵測方式 | 規則制（6 種） | 無監督 ML + Graph |
| 適合場景 | 快速查一筆 tcpdump | 持續監控 / Honeypot |
| 告警 | 無 | 可接 TG / ELK |

兩個不是替換關係——HTML 版適合臨時查，Plus 適合長期跑。

---

## 限制

- Input 只吃 tcpdump 文字輸出（`-tttt -nn`），不直接讀 `.pcap`
- LightGBM 每次重訓，Flow 數量很少時（< 4）退化為加權平均
- Graph 分析假設所有 Flow 在同一時間窗內——長時間的低速掃描可能被分散到不同窗口而漏掉
- 沒有內建告警，需要自己接 TG / ELK

---

## Repo

[github.com/kerr20801/PacketScope](https://github.com/kerr20801/PacketScope)

HTML 版（`index.html`）+ ML 版（`src/`）在同一個 repo，場景不同，各取所需。
