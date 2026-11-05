# FortiGate 型號能撐多少 VPN——企業角度的容量與拓撲設計

結論先講：

> **DH Group、加密演算法這些「怎麼設」的問題，網路上到處查得到答案。真正決定企業 VPN 架構好不好用的，是「這台設備撐不撐得住」——而這件事要對照官方 datasheet 的硬體規格，不能只看安全強度表。**
>
> 常見的錯誤示範：把 SSL-VPN + 多條 IPsec S2S 全部疊在同一台中低階設備上，然後才發現瓶頸不在加密演算法，是在那台設備從一開始的容量就不夠。

本文分兩塊：**容量**（三個常見型號能撐多少、一個真實拓撲下誰是瓶頸）跟**協定**（Group 14~32 怎麼選、IKEv1 該不該留）。加密演算法與 Rekey 的實戰陷阱另一篇 [IPsec VPN 實戰配置與避坑指南](ipsec_vpn_enterprise_guide.md) 已經寫得很細，本文不重複，只補它沒有的：**型號規格對照**與**Group 27~32 全表**。

---

## 一、三個型號能撐多少——官方 datasheet，不是安全強度表

企業最常見的組合是：分支用 **101F**，區域中心用 **201G**，總部或資料中心用 **401F**。三者的差距不是線性的，是加密晶片本身就不同代：

| | **101F** | **201G** | **401F** |
|---|---|---|---|
| 加密晶片 | SOC4 | NP7Lite (SP5) + CP10 | NP7 + CP9 |
| IPS Throughput | 2.6 Gbps | 9 Gbps | 12 Gbps |
| NGFW Throughput | 1.6 Gbps | 7 Gbps | 10 Gbps |
| **IPsec VPN Throughput**（512B, AES256-SHA256）| **11.5 Gbps** | **36 Gbps** | **55 Gbps** |
| Gateway-to-Gateway IPsec 隧道數上限 | 2,500 | 2,000 | 2,000 |
| Client-to-Gateway IPsec 隧道數上限 | 16,000 | 16,000 | 50,000 |
| SSL-VPN Throughput | 1 Gbps | 3 Gbps | 3.6 Gbps |
| **SSL-VPN 建議併發數上限**（Tunnel Mode）| — | **500** | **5,000** |
| Concurrent Sessions (TCP) | 1.5M | 11M | 7.8M |
| Firewall Policies | 10,000 | 10,000 | 10,000 |

出處：[FortiGate 100F Series Datasheet](https://www.fortinet.com/content/dam/fortinet/assets/data-sheets/pdf/fortigate-100f-series.pdf)、[FortiGate 200G Series Datasheet](https://www.fortinet.com/content/dam/fortinet/assets/data-sheets/fortigate-200g-series.pdf)（Rev.11，2026-07-09）、[FortiGate 400F Series Datasheet](https://www.fortinet.com/content/dam/fortinet/assets/data-sheets/fortigate-400f-series.pdf)（Rev.22，2026-04-14）。**datasheet 會改版，這幾個數字建議每次動手前重抓一次官網原檔**，不要憑印象或憑舊筆記。

看這張表要注意兩件事：

1. **IPsec 隧道數不是隨型號等比放大。** Gateway-to-Gateway 隧道數 101F（2,500）反而比 201G/401F（2,000）多——別以為越貴的型號每項數字都比較大，要對照的是你實際用得到的那一欄。
2. **SSL-VPN 併發數上限差 10 倍**（201G 的 500 vs 401F 的 5,000）。這一欄，才是下一節那個真實拓撲的關鍵。

---

## 二、為什麼 hub 最累——一個真實的三機拓撲

常見的企業拓撲長這樣：總部一台 **201G**，同時扛：

- **SSL-VPN**，給員工遠端連線
- **一條 IPsec S2S 到 101F**（分支）
- **一條 IPsec S2S 到 401F**（資料中心）
- **101F 跟 401F 彼此不相通**——兩個 spoke 之間沒有隧道，所有流量都要繞經 201G 這個 hub

```
                 ┌─────────────┐
   SSL-VPN ──────│    201G     │────── S2S ──── 401F（資料中心）
   （員工）       │  （HQ hub）  │
                 └──────┬──────┘
                        │
                       S2S
                        │
                    101F（分支）
```

*（IP 均為 RFC 1918 示意用途，非實際部署位址）*

這個拓撲下，**201G 是全公司唯一一台要同時做三件事的設備**，而 101F/401F 各自只需要應付「跟 201G 對接」這一件事。對照第一節的表就能看出問題所在：

| 角色 | 這台要扛的事 | 對照規格 |
|---|---|---|
| 201G（hub）| SSL-VPN + 2 條 IPsec S2S 同時進行 | SSL-VPN 建議併發**只有 500** |
| 401F（spoke）| 只需應付跟 201G 那一條 S2S | SSL-VPN 上限 5,000，但這個角色用不到 |
| 101F（spoke）| 同上 | IPsec throughput 11.5 Gbps 對分支已經綽綽有餘 |

**同一顆「同時做 SSL-VPN + 兩條 S2S」的角色,擺在 401F 上完全不會是瓶頸,擺在 201G 上就非常吃緊**——201G 的 SSL-VPN 併發上限只有 401F 的十分之一。這不是配置問題，是硬體等級選錯了位置：**流量匯聚的那個節點，理當放最高階的設備，而不是照組織階層（總部用中階、分支用低階）去配置。**

實務上兩個方向可以調：

1. **把 SSL-VPN 從 hub 移走**——如果公司有資源，讓 401F（資料中心）兼職扛 SSL-VPN，201G 專心做 S2S 匯聚。
2. **限制 rekey 頻率降低 hub 的協商負擔**——DH Group 若用傳統 MODP（Group 14），每次 Phase1 rekey 都要算一次昂貴的大質數模冪運算；三條連線（SSL-VPN + 2× S2S）同時在跳 rekey 時，疊加在本來就吃緊的 201G 上會很明顯。這點在下一節細講。

---

## 三、101F 對 401F 的 S2S 該怎麼配

跨型號對接，遵守**木桶效應**：兩端協商出來的 Phase1/Phase2 proposal，最終效能上限是**較弱那端**決定的，不是取平均、更不是取較強那端。101F 對 401F，101F 就是那個木桶最短的板子。

實務作法：

- **Proposal 用 101F 撐得住的設定**，401F 端不需要為了「反正比較強」就開更重的參數（例如 Group 21 或 4096-bit MODP）——101F 算不動一樣會拖垮協商。
- **確認 101F 的 SOC4 有沒有硬體加速你要用的演算法。** 若沒有，加解密會退回 CPU 處理，在多隧道或大流量下會導致 CPU 滿載、封包延遲，甚至系統不穩定——這比選錯 DH Group 更容易在生產環境炸掉。加密演算法選擇與 rekey 陷阱的完整清單見 [IPsec VPN 實戰配置與避坑指南](ipsec_vpn_enterprise_guide.md) 第 4、5 節。

---

## 四、DH Group 14~32 全表

| Group | 類型 | 依據 RFC |
|---|---|---|
| 14~18 | MODP（傳統模指數）| RFC 3526 |
| 19~21 | ECP（NIST 橢圓曲線）| RFC 5903（IETF 標準定義；FortiOS 7.4 官方文件以 RFC 6989 間接佐證支援）|
| **27~30** | **Brainpool ECP** | **RFC 6954** |
| **31 / 32** | **Curve25519 / Curve448** | **RFC 8031** |

出處：[FortiOS 7.4 Supported RFCs](https://fortinetweb.s3.amazonaws.com/docs.fortinet.com/v2/attachments/66196a6a-db05-11ed-8e6d-fa163e15d75b/FortiOS-7.4-Supported_RFCs.pdf)（2025-01-23 版），7.4.x 全系列適用，涵蓋到 7.4.12。**RFC 3526 / 6954 / 8031 三條是該文件逐字列出的**；19~21 這行的 RFC 5903 是 IETF 定義這幾個 Group 的通用標準，該份文件本身沒有直接點名，只透過 RFC 6989（Additional Diffie-Hellman Tests）間接佐證。

Group 14 沒有壞掉、也還算安全，但它是「相容性最大公約數」，不是理想值。**DH 運算只發生在 Phase1 協商/rekey 的當下,不影響穩態吞吐量**——真正影響穩態吞吐量的是加密演算法（下段）。所以換 Group 省的是「rekey 那一瞬間的 CPU 負擔」，這件事在第二節那種多連線同時 rekey 的 hub 上特別有感。

實務寫法：不要把 Group 寫死成一個值，讓兩端協商出雙方都支援的最強選項：

```
set dh-group 21 20 19 14      ← 自家 FortiGate 對打，21/20/19 優先，14 保底相容第三方設備
set proposal aes256gcm sha256 ← AEAD，不要 CBC+SHA256 分開算
set ike-version 2
```

### 比選 Group 更值得做的一件事：RFC 8784

FortiOS 7.4 支援 **RFC 8784（Mixing Preshared Keys in IKEv2 for Post-quantum Security）**——PPK（Post-quantum Preshared Key）。做法是在標準 IKEv2 金鑰交換之外，額外混入一把預共用金鑰，讓即使未來 DH 交換被量子電腦破解，連線仍有一層對稱式金鑰保護。這**不是換 Group 的問題，是額外加一層**，7.4.x 就能直接用。相較於在 Group 14 跟 19 之間反覆糾結，這才是真正面向未來的動作。

---

## 五、IKEv1 vs IKEv2：官方文件已經用行動表態

FortiOS 7.4 Supported RFCs 的 Cryptography 分類列了將近 20 條 IKEv2 相關 RFC——訊息分片（RFC 7383）、Session Resumption（RFC 5723）、ECDSA 認證（RFC 4754）、TCP 封裝（RFC 8229，僅限 IKEv2）——**但完全沒有列出 IKEv1 的 RFC**（沒有 RFC 2409、RFC 2408）。

這不代表 IKEv1 從 CLI 消失了（`set ike-version 1` 還在），但官方文件的重心已經完全放在 IKEv2。**沒有跨廠牌老設備相容性包袱的話，新建的每一條 S2S 都該用 IKEv2**，理由不只是這裡列的協商效率跟 NAT-T，更直接的訊號是：Fortinet 自己往哪個方向持續投入功能，看這份文件就知道。

---

## 小結

| 決策 | 依據 |
|---|---|
| SSL-VPN 這種高併發角色，該放在哪台 | 對照 datasheet 的「Concurrent SSL-VPN Users」，不是看組織階層 |
| 101F 對 401F 怎麼配 proposal | 木桶效應，以 101F 的硬體加速能力為準 |
| DH Group 選多少 | 自家設備對打用 19/20/21，第三方/老設備保底用 14；別浪費力氣在 Group 14 vs 19 上，RFC 8784 才是真正的下一步 |
| IKEv1 還要不要留 | 除非有老設備相容性需求，否則新隧道一律 IKEv2 |

延伸閱讀：[IPsec VPN 實戰配置與避坑指南](ipsec_vpn_enterprise_guide.md)（Rekey 陷阱、GCM 配置細節）、[IKEv1 對 IKEv2 通訊機制與 CLI 逐項對照](ikev1-vs-ikev2-mechanics-cli.md)、[用 IPsec Aggregate 做安全的 Phase1 切換](ipsec-aggregate-safe-rollout.md)（怎麼把本文的建議實際搬上正式環境）
