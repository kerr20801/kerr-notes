# IKEv1 對 IKEv2：通訊機制與 FortiOS 設定逐項對照

結論先講：

> **新建的每一條隧道，沒有相容性包袱就用 IKEv2。** 差距不是「比較新比較安全」這種空話，是協商步驟少一半、NAT-T 是規格內建不是外掛、訊息分片是原生功能不是廠商私有擴充。這篇把這些差異攤開講清楚，並附 FortiOS CLI 逐項對照表。
>
> **認證後端（LDAP/EAP）的相容性陷阱不在本文**——那是結構性問題，另有 [〈同一個 AD 群組，IKEv1 連得上、IKEv2 連不上〉](2026-08-07-fortigate-ikev2-eap-ldap.md) 整篇在講，本文只補它沒涵蓋的通訊機制層。
>
> ⚠️ **上面那句「沒有相容性包袱就用 IKEv2」有一個已實測到的例外：ADVPN hub。** 一條 phase1 底下多條 SA、且各 spoke 隧道位址靠交換學來的情況下，改成 IKEv2 會導致 hub 解析不出該用哪條 SA——控制平面全綠、資料平面全斷。詳見 [〈隧道 up、BGP Established、路由也學到了——封包就是過不去〉](2026-08-20-advpn-ikev2-tunnel-id-unresolved.md)。一般點對點 S2S 不受影響（只有一條 SA，沒有選擇題）。

---

## 一、訊息交換流程：協商步驟差在哪

IKEv1 有兩種交換模式，IKEv2 只有一種基本交換。畫出來看差在哪：

```
IKEv1 Main Mode（6 則訊息，3 個往返）
  發起端 → 回應端   HDR, SA
  回應端 → 發起端   HDR, SA
  發起端 → 回應端   HDR, KE, Ni
  回應端 → 發起端   HDR, KE, Nr
  發起端 → 回應端   HDR*, IDi, HASH_I         ← 身分在第 5 則才出現，此時已加密
  回應端 → 發起端   HDR*, IDr, HASH_R

IKEv1 Aggressive Mode（3 則訊息，1.5 個往返）
  發起端 → 回應端   HDR, SA, KE, Ni, IDi       ← 身分在第 1 則就送出，明文
  回應端 → 發起端   HDR, SA, KE, Nr, IDr, HASH_R
  發起端 → 回應端   HDR, HASH_I

IKEv2 基本交換（4 則訊息，2 個往返，寫死在 RFC 7296，沒有模式可選）
  發起端 → 回應端   HDR, SAi1, KEi, Ni          ← IKE_SA_INIT
  回應端 → 發起端   HDR, SAr1, KEr, Nr, [CERTREQ]
  發起端 → 回應端   HDR, SK{IDi, [CERT,] AUTH, SAi2, TSi, TSr}   ← IKE_AUTH，此時已加密
  回應端 → 發起端   HDR, SK{IDr, [CERT,] AUTH, SAr2, TSi, TSr}
```

三個看得出來的差異：

1. **IKEv2 沒有「模式」可選**——不像 IKEv1 要在 main（安全但慢）跟 aggressive（快但身分明文）之間權衡，IKEv2 的基本交換本身就是「先建立加密通道，身分驗證在通道裡進行」，等於是把 main mode 的安全性用更少的訊息數做到。
2. **Aggressive Mode 的身分在第一則訊息就明文送出**——這是[〈同一個 AD 群組，IKEv1 連得上、IKEv2 連不上〉](2026-08-07-fortigate-ikev2-eap-ldap.md)裡「aggressive mode 會外洩 PSK 雜湊」說法的根源，也是該篇實測推翻它、改用 main mode 的原因。
3. **Phase 2（IPsec SA 協商）在 IKEv2 裡被併進同一輪交換**（`SAi2`/`SAr2` 出現在 `IKE_AUTH` 訊息裡），IKEv1 則是 Phase 1 完成後另外跑一輪 Quick Mode。等於 IKEv2 把 IKEv1 的「Phase1 3輪 + Phase2 3則」壓縮成 2 輪就把兩個 SA 都談完。

---

## 二、三個協定層差異，附 RFC 佐證

| | IKEv1 | IKEv2 | 依據 |
|---|---|---|---|
| **NAT 穿透** | 需要額外協商擴充（`set nattraversal enable`，靠 RFC 3947 的偵測機制外掛上去）| 規格內建，不需要額外協商 | RFC 3947（v1 擴充）vs RFC 7296（v2 本體含 NAT 偵測）|
| **訊息分片** | 無標準機制，各廠牌各自實作（FortiGate 有自己的分片邏輯，跟其他廠牌不保證互通）| **RFC 7383，原生標準功能** | FortiOS 7.4 官方 [Supported RFCs](https://fortinetweb.s3.amazonaws.com/docs.fortinet.com/v2/attachments/66196a6a-db05-11ed-8e6d-fa163e15d75b/FortiOS-7.4-Supported_RFCs.pdf) 明確列出 RFC 7383 |
| **SA Rekey 碰撞** | Phase2 沒有內建的碰撞偵測，兩端同時發起 rekey 時可能各自建出一條新 SA、舊 SA 各自逾時，短暫出現重複或懸空的 SA | `CREATE_CHILD_SA` 交換帶明確的 nonce 與 message ID，同一組 SA 的 rekey 有序列化機制 | RFC 7296 §2.8 |
| **失聯偵測** | DPD（Dead Peer Detection），廠商擴充（RFC 3706）| 同樣可用 DPD；另有 **MOBIKE**（RFC 4555，位址變動時不必重建整條 SA）| RFC 3706（v1/v2 通用）；MOBIKE 僅 v2 |

**Rekey 碰撞這條要老實講**：IKEv2 的機制設計上確實比 IKEv1 更不容易產生碰撞，但「完全解決」是過度簡化的說法——實務上仍然可能因為兩端幾乎同時判斷需要 rekey 而各自起頭，只是 IKEv2 的訊息結構讓後續協商比 IKEv1 更容易收斂，不是保證零碰撞。如果你的環境 rekey 間隔設得很短、且兩端時鐘/lifetime 幾乎同步，這仍然值得留意，不要當成「換了 IKEv2 這件事就不用管了」。

---

## 三、FortiOS `phase1-interface` 逐項對照

```
config vpn ipsec phase1-interface
    edit "example"
        set ike-version {1 | 2}
        ...
    next
end
```

| 參數 | IKEv1 | IKEv2 | 說明 |
|---|---|---|---|
| `mode` | `aggressive` / `main` | 不適用（沒有模式可選）| v2 設這個欄位會被忽略 |
| `xauthtype` | 可設（XAuth 額外認證輪）| 不適用 | v1 專屬的使用者認證擴充 |
| `eap` | 不適用 | `enable` / `disable` | v2 專屬，EAP 認證框架，後端相容性見另一篇 |
| `authusrgrp` | 可設（搭配 XAuth）| 可設（搭配 EAP）| 兩邊都能綁使用者群組，但底層認證機制完全不同 |
| `fragmentation` | 可設，FortiGate 私有分片邏輯 | 可設，走 RFC 7383 標準分片 | 同一個 CLI 關鍵字，底層機制不同，見上表 |
| `nattraversal` | `enable`（額外協商）/ `disable`/ `forced` | 同樣可設，但屬於規格本體，不是外掛 | |
| `dpd` | 可設 | 可設 | 兩邊通用 |
| `dhgrp` | 最多可設 3 組（超過會回 `too many argument`）| 同樣的欄位限制 | 這條上限值是 EAP/LDAP 那篇實測踩出來的，兩個 IKE 版本共用同一個底層限制 |

**沒有列在表裡的欄位（`proposal`、`peertype`、`localid`、`remote-gw` 等）兩個版本語法完全一致**，跨版本切換時不用重寫整段設定，只需要處理上表這幾項。

---

## 四、決策矩陣

| 情境 | 建議 |
|---|---|
| 全新的 S2S，兩端都是近代設備 | IKEv2，沒有理由留 v1 |
| Client-to-Gateway，後端是 LDAP/AD、要用群組管理權限 | **先留 IKEv1 + XAuth + main mode**，見[同一個 AD 群組那篇](2026-08-07-fortigate-ikev2-eap-ldap.md)；要轉 v2 得先加一層 RADIUS |
| 對接老舊第三方 VPN 設備，對方不支援 v2 | 沒得選，只能 v1 |
| 需要行動裝置漫遊時不斷線（Wi-Fi ↔ 行動網路切換）| 只有 IKEv2 有 MOBIKE，v1 做不到 |
| 隧道要跨 NAT，且環境本身 NAT 規則複雜 | IKEv2，NAT-T 是規格本體，除錯時少一層「這是不是額外協商沒談成」的變數 |

---

延伸閱讀：[FortiGate 型號能撐多少 VPN——企業角度的容量與拓撲設計](fortigate-model-capacity-vpn-topology.md)（第五節有 IKEv1 未出現在官方 RFC 清單的佐證）、[同一個 AD 群組，IKEv1 連得上、IKEv2 連不上](2026-08-07-fortigate-ikev2-eap-ldap.md)（認證後端相容性的完整除錯過程）、[用 IPsec Aggregate 做安全的 Phase1 切換](ipsec-aggregate-safe-rollout.md)（本文的 IKEv1→IKEv2 建議怎麼安全搬上正式環境）
