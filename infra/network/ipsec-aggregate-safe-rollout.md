# 用 IPsec Aggregate 做安全的 Phase1 切換：DH Group 跟 IKE 版本不能用同一招

結論先講：

> **Phase1 的參數不是全部都能「先加後驗證再收尾」地漸進式切換。DH Group 可以，IKE 版本跟 Mode 不行——因為前者是清單，後者是單一值，兩端沒對上就是直接談判失敗，沒有退路。**
>
> 這篇要講的是：這個差異怎麼影響正式環境的切換順序，以及 **IPsec Aggregate（多條隧道綁成一組互為備援）如何把「單一值、沒有退路」的參數也變成可以安全逐條硬切**。

前情提要：容量規格見 [FortiGate 型號能撐多少 VPN](fortigate-model-capacity-vpn-topology.md)，通訊機制與 CLI 對照見 [IKEv1 對 IKEv2](ikev1-vs-ikev2-mechanics-cli.md)。本文只講「怎麼把上面兩篇的建議實際搬上正式環境」。

---

## 一、兩種參數，兩種切換手法

| | `dh-group` | `ike-version` / `mode` |
|---|---|---|
| 值的形狀 | **清單**，可以多值並存（`set dh-group 19 14`）| **單一值**，只能設一個 |
| 兩端沒完全對上時 | 自動協商到雙方都支援的最強共同值 | 直接協商失敗，沒有退路 |
| 安全切換手法 | **先加、驗證談到新值、再拿掉舊值**（三步驟，過程中隨時可退回舊值頂著）| **沒有中繼態**，改了就是改了，兩端必須同時對齊 |

這個差異很容易被忽略，因為兩者都是同一個 `config vpn ipsec phase1-interface` 底下的欄位，改起來手感一樣，但**背後的容錯機制完全不同**。把 DH Group 那套「先加後驗證」的直覺套用到 IKE 版本或 Mode 上，會誤以為兩端只要有一邊沒改到就會像 DH Group 一樣自動退回舊值頂著——**不會，它會直接斷**。

---

## 二、S2S 通常不需要 aggressive mode 這個中繼站

Aggressive mode 存在的常見理由：**同一個介面上掛了多條、各自不同 PSK 的 dial-up 隧道時，main mode 沒辦法在還沒認出對方身分前選出該用哪把 PSK**，只好讓身分在第一則訊息就明文送出，gateway 才有得選。

但 **S2S（site-to-site）通常沒有這個限制**——兩端都是固定的 peer IP，不是一堆 PSK 混在同一介面的 dial-up 情境。動手前先確認：

```
show vpn ipsec phase1-interface
    看 remote-gw 是不是固定 IP，不是 0.0.0.0（動態）
```

如果是固定 IP，main mode（甚至直接跳過 main、上 IKEv2）通常沒有結構性阻礙，現在還在 aggressive 多半是沿用舊範本的預設值，不是刻意的技術決定。

**既然要編輯 phase1-interface 這個檔案，就一次到位**，不要 aggressive→main、以後哪天想升級又要重開一次維護窗口去改 IKE 版本。同一次編輯視窗，`ike-version` 跟 `dh-group` 一起改：

```
edit "member1"
    set ike-version 2       ← 一次到位，IKEv2 沒有 mode 這個概念，順便解決掉
    set dh-group 19 14      ← 這個可以漸進式，19 優先、14 保底
next
```

---

## 三、IPsec Aggregate 把「沒有退路」的參數也變得可以安全切

沒有備援隧道時，任何「沒有退路」的硬切（ike-version、mode）都有全斷風險——這時只能靠參數本身能不能玩漸進式來避險，而 `dh-group` 可以、`ike-version`/`mode` 不行，所以後者只能挑維護窗口、兩端同時對齊、賭一次。

**有 IPsec Aggregate 之後，切換單位從「全部隧道」變成「單一 member」**：

```
config system ipsec-aggregate
    edit "agg1"
        set member "member1" "member2" "member3"
    next
end
```

三條 member 互為備援，切換節奏變成：

```
member1  →  硬切 ike-version 2 + dh-group 19 14  →  斷線期間 member2/3 頂著
              ↓ 確認 member1 重談成功、AGG 把它算回可用池
member2  →  同上
              ↓
member3  →  同上
```

**每次只有一條 member 在空窗期，其他兩條扛著整體連通性**。這讓「沒有退路」的 `ike-version` 硬切也能安全地逐條做完，不需要挑半夜維護窗口賭兩端同時對齊。

### 更進一步：拿一條 member 當真實硬體測試，不用猜

這招不只是拿來避險，還可以拿來**驗證你原本沒把握的東西**。舉例：某個 DH Group（例如 31，Curve25519）在你其中一台設備的加密晶片上到底有沒有硬體加速，官方文件不一定查得到明確答案。有 AGG 的話，直接挑一條 member 上那個 Group，觀察它的表現：

```
diagnose vpn ike gateway list
    確認那條 member 實際協商出來的值，是不是真的談到你設的那個（不是靜默退回別的）

diagnose vpn tunnel list
    看那條 tunnel 的加密是不是走硬體卸載，不是退回 CPU 軟解
```

如果那條 member 協商正常、也確認吃到硬體加速，代表可以把其他 member 也跟進；如果它退回軟解或協商不穩，代表在硬體世代統一之前，那個參數只留給已經驗證過的設備。**全程只有一條 member 承擔這個未知數，出問題不影響整體。** 這比查文件猜測，或是全部設備一次上去賭，可靠得多。

---

## 四、正式環境的 rollout 順序

給一個典型三條 member 的完整順序：

| 階段 | 動作 | 風險範圍 |
|---|---|---|
| 1 | member1：`ike-version 2` + `dh-group 19 14` | member1 空窗，member2/3 頂著 |
| 2 | 驗證 member1 重談成功、確認吃到硬體卸載 | — |
| 3 | member2：同上 | member2 空窗 |
| 4 | member3：同上 | member3 空窗 |
| 5 | 全部 member 確認都談到 19 之後，等幾天觀察 | 無風險，純觀察期 |
| 6 | 逐條把 `dh-group` 收尾成單一值 `19`（拿掉 14）| 這步驟本身安全，因為此時兩端已經都在 19，收掉 14 不影響現有協商 |

第 6 步之所以安全，是因為 `dh-group` 是清單型參數——拿掉一個仍在協商用的值不會中斷正在跑的 SA，下次 rekey 才會用新的提案清單重談。這跟第 1 步的硬切完全是两回事，不要用同一種緊張程度去對待。

---

延伸閱讀：[FortiGate 型號能撐多少 VPN](fortigate-model-capacity-vpn-topology.md)、[IKEv1 對 IKEv2 通訊機制與 CLI 逐項對照](ikev1-vs-ikev2-mechanics-cli.md)
