# FortiOS 7.6 拿掉了 SSL VPN，我被迫遷到 IPSec —— 順便挖出「隧道通了但什麼都不通」的三種死法

寫於 2026-08-04

---

FortiOS 7.6 在低階機型（2GB RAM，60F 這類）**移除了 SSL VPN tunnel mode**。官方立場很明確：改用 IPSec。

我的 60F 從 7.4.12 升到 7.6.7，本來以為是例行升版，結果變成一次完整的 VPN 架構遷移。過程中撞到三個「隧道明明建起來了，但就是不通」的情況——**而且三個的根因完全不同，症狀卻長得一模一樣**。

這篇不講「怎麼設定 IPSec VPN」（官方文件寫得很清楚），只講那些**設定看起來完全正確、但實際上不會動**的地方。

> 文中所有 IP、網域已改為範例值。

---

## 先講升版本身：比預期無聊

| | 升版前 | 升版後 |
|---|---|---|
| 記憶體 | 57~58% | **51%** |
| conserve mode | 從未進入 | 仍為 0 |

**記憶體不升反降**。我原本擔心 2GB 機型跑 7.6 會很緊，結果移除 SSL VPN tunnel 釋出的資源比新版本多吃的還多。設定也全數存活，UTM profile 一個沒掉。

有一件事值得先澄清，因為我自己一開始就搞錯：**7.6 移除的是 SSL VPN 的 tunnel mode，不是整個 SSL VPN**。web mode（瀏覽器 portal）還在。

驗證方法比查文件可靠——直接把 config 物件的欄位列出來看：

```bash
curl -sk "$FGT/api/v2/cmdb/vpn.ssl.web/portal/full-access" -H "Authorization: Bearer $TOKEN" \
  | python3 -c "import json,sys; print(sorted(json.load(sys.stdin)['results'][0].keys()))"
```

升版後這個物件裡 **tunnel / split / ip-pool 相關欄位全部消失**，但 `web-mode`、`bookmark-group` 都在。欄位在不在，比任何版本說明都準。

---

## 死法一：「Phase 2 不通」其實根本沒壞

我有一條 ADVPN（Auto-Discovery VPN，hub-and-spoke 架構）。升版後的症狀：

> Phase 1 建得起來，Phase 2 看起來不通，是版本問題嗎？

查下去發現 **Phase 2 一直是好的**：

```
Advpn_0 → phase2 status=up   已連 28 分鐘   14.8KB in / 8.2KB out
Advpn_1 → phase2 status=up   已連 80 分鐘   63.5KB in / 23.1KB out

event/vpn log 裡「IPsec phase 2 error」：0 筆
```

**誤判是怎麼來的**：GUI 上那個叫 `Advpn` 的項目永遠顯示 0 連線。因為它是 **dial-up 的範本，不是實際隧道**。真正的連線是 `Advpn_0`、`Advpn_1` 這些帶編號的子項目。看範本的狀態去判斷隧道死活，會得到完全相反的結論。

### 真正壞的是 BGP，而 BGP 壞在一個你不會想到的地方

```
BGP 鄰居：4 個全部卡在 Active / Connect，沒有一個 Established
BGP 學到的路由：0 筆
```

隧道通、BGP 不通 → 學不到對端網段 → 沒有路由 → 使用者體感就是「不通」。

而 BGP 起不來的原因是：

```
Advpn 隧道介面的 IP：0.0.0.0
BGP router-id：10.255.1.1
```

**BGP 是 TCP 179 的協定，需要一個來源 IP 才能建立連線。** 隧道介面是 `0.0.0.0` 等於這條隧道在路由層沒有身分，封包根本發不出去，只能永遠卡在 Active 重試。

修法只有三行：

```
config system interface
    edit "Advpn"
        set ip 10.255.1.1 255.255.255.255
        set remote-ip 10.255.1.254 255.255.255.0
    next
end
```

改完 60 秒內 BGP 就 Established，路由也學進來了。

- `ip ... /32` — hub 在隧道上的位址，**必須跟 BGP `router-id` 一致**。用 /32 是因為這是點對點虛擬介面。
- `remote-ip ... /24` — **這行的遮罩才是重點**。它定義「這條隧道的對端網段是 10.255.1.0/24」，讓 BGP 的 `neighbor-range`（動態接受 spoke）對得上。

> 副作用：改介面 IP 會讓隧道重新協商，對端會自動重連，但當下有流量會斷一瞬間。

### 這裡的通則：FortiGate 的 IPsec 是兩個獨立的層

| 層 | GUI 位置 | 負責什麼 |
|---|---|---|
| **加密隧道**（phase1/phase2） | VPN → IPsec Tunnels | 對端是誰、怎麼加密、SA 建不建得起來 |
| **虛擬網路介面** | Network → Interfaces | 這條隧道在**路由層**的身分（有沒有 IP） |

**Phase 1/2 只負責「封包能不能安全送到對面」，完全不管「這條隧道有沒有 L3 位址」。**

所以會出現「隧道 up、雙向有流量、log 乾淨，但上面跑的任何協定都動不了」。

**為什麼特別容易漏**：用 VPN Wizard 建的隧道，FortiGate 會自動配隧道 IP；**手動建 phase1/phase2（CLI 或 Custom 模式）時，介面 IP 預設就是 `0.0.0.0`，要自己去 Interfaces 補**。

兩個頁面各自看都「設定完整」，問題出在它們中間——這是我認為 FortiGate 最容易讓人栽跟頭的結構設計。

---

## 死法二：客戶端相容性才是主戰場

SSL VPN 沒了之後要讓所有裝置改連 IPSec。這時才發現，**限制不在防火牆，在客戶端**。

實測整理：

| 客戶端 | IKEv1 aggressive + XAuth | IKEv2 + PSK | IKEv2 + EAP |
|---|---|---|---|
| macOS 原生 | ✅（Cisco IPSec 選項） | ⚠️ | ✅ |
| **Windows 11 原生** | ❌ **完全不支援** | ❌ **不支援 PSK** | ✅（需伺服器憑證） |
| iOS 原生 | ✅ | ⚠️ | ✅ |
| Android 原生 | ❌ | ✅ | ✅ |
| FortiClient | ✅ | ✅ | ✅ |

幾個容易誤解的點：

**IKEv1 和 IKEv2 是兩套完全不同的協定**，不是「版本高低可以往下相容」。封包格式、交換流程、狀態機都不同，**沒有協商降級這回事**。伺服器設 IKEv1，客戶端就只能選 IKEv1，選錯會在第一個封包就被打回：

```
reason: peer SA proposal not match local policy
cookies: 8d09e19f1a4222f4/0000000000000000
                          ^^^^^^^^^^^^^^^^ responder cookie 全 0 = 連第一輪都沒過
```

**Windows 11 原生 VPN 沒有「Cisco IPSec」這種選項**。它只做 IKEv2、L2TP/IPSec（IKEv1 **main** mode）、SSTP、PPTP。所以 IKEv1 aggressive + XAuth 這種最常見的 FortiGate dial-up 設定，Windows 原生**永遠連不上**，只能裝 FortiClient 或改用 IKEv2。

而 macOS 有 Cisco IPSec 選項，所以會出現「Mac 連得上、Windows 連不上」這種看起來很玄的狀況——其實是結構性差異，不是設定歪掉。

### 好消息：同一個介面可以並存多條

FortiGate 靠「IKE 版本 + 交換模式」區分進來的連線，所以：

```
Advpn           IKEv1 main mode        ← ADVPN hub
IPSEC_LEGACY    IKEv1 aggressive mode  ← macOS / FortiClient
IPSEC_IKEV2     IKEv2                  ← 原生用戶端
```

三條共存互不干擾。這讓「**並存過渡**」變得可行——不用一次切換、沒有斷線風險。

在 SSL VPN 已經消失、IPSec 是唯一遠端管道的情況下，這點很重要：**不該用「改一條、全部人一起賭」的方式動它**。

---

## 死法三：IKEv2 建起來了，但流量是 0

這是最花時間的一段，因為連續撞到四個**互相獨立**的問題，每修好一個就露出下一個。

### ① EAP 拿客戶端的 IP 當帳號

症狀：phase 1 協商成功好幾輪，然後 failure → SA 刪除 → 重試。體感是「一直轉、連不上」。

關鍵證據在 `event/vpn` log：

```
user: 10.177.194.102       ← FortiGate 拿去驗證的「帳號」
cookies: 82ac.../5ea5...   ← 雙方 cookie 都非零 = 加密協商已過，卡在認證
locport: 4500              ← NAT-T 正常
```

那個 `user` 欄位是手機的**內網 IP**。FortiGate 拿這串去使用者群組裡找帳號，當然找不到。

根因是 `eap-identity` 的模式：

| 值 | 行為 |
|---|---|
| `use-id-payload` | 拿客戶端的 IKE ID payload 當帳號 |
| `send-request` | **主動送 EAP Identity Request，要客戶端提供帳號** |

手機送的 IKE ID 是它的內網 IP，所以 `use-id-payload` 就抓到了那串數字。改成 `send-request` 即解。

### ② 政策的來源位址還指著舊的 IP 池

我把 mode-cfg 的 IP 池換了範圍（避免和另一條 VPN 重疊），但**忘了同步政策引用的位址物件**：

```
客戶端實際拿到：10.99.1.1
policy 允許來源：10.99.2.11 ~ 10.99.2.20   ← 舊池
```

這裡有個好用的判斷技巧：

- **`in=0 out=0` 雙向全 0** → 封包在**政策層**就被丟掉，或客戶端根本沒送
- **有 out 沒 in** → 路由/回程問題
- traffic log 裡**連 `action=deny` 都沒有** → 封包根本沒進 FortiGate

我那次是雙向全 0 + log 完全沒紀錄，直接指向「政策不匹配」。

### ③ 內部 DNS 把 VPN 客戶端當成外部使用者

內部 DNS（BIND）用的是 split-horizon view：

```
acl internal { 10.0.0.0/16; }      ← VPN 池 10.99.1.x 不在裡面
view "internal" → 內網 zone（10.0.x）
view "external" → 公網 zone         ← VPN 客戶端落到這裡
```

**即使 DNS 查詢送達了，客戶端拿到的也是公網記錄而不是內網 IP。**

這個坑很隱蔽，因為「DNS 有回應」——只是回錯的答案。修法是把 VPN 池加進 `internal` ACL，用 `rndc reconfig` 套用（不中斷服務）。

### ④ 手機根本不把 DNS 查詢送進 VPN

前三個修完，內網流量通了，但**用主機名稱還是連不到**。DNS 伺服器上**零筆**來自 VPN 客戶端的查詢紀錄。

根因是 split tunnel 的作業系統行為：**iOS/Android 只把「符合指定網域」的查詢送給 VPN 的 DNS，其餘一律用行動網路自己的 DNS。**

FortiGate 有三個名字很像的 DNS 欄位，這是最容易搞錯的地方：

| 欄位 | 作用 | 適用 |
|---|---|---|
| `domain` | 單一預設 DNS 網域 | **Cisco Unity 擴充＝IKEv1 專用**。`unity-support disable` 時設了會被直接忽略（而且 API 不會報錯） |
| `dns-suffix-search` | 解析短名稱時自動補後綴 | 兩者皆可 |
| **`internal-domain-list`** | **這些網域必須用 VPN 的 DNS 查（split-DNS）** | **IKEv2 的正解**，對應標準的 `INTERNAL_DNS_DOMAIN` config 屬性 |

```
config vpn ipsec phase1-interface
    edit "IPSEC_IKEV2"
        set internal-domain-list "example.com"
    next
end
```

補上之後，DNS 伺服器立刻看到查詢進來，而且落在正確的 view：

```
client 10.99.1.1#55585 (app.example.com): view internal: query: app.example.com IN A
                                          ^^^^^^^^^^^^^ 拿到內網 IP ✅
```

---

## 一個會讓你懷疑人生的 API 行為

FortiGate 的 REST API **對某些欄位會回 `success`，但實際上完全沒寫入**，而且不報任何錯。

我連續中了三次才發現規律：**`member_table` 型別的欄位不能用父層 PUT 設定**，要用子端點 POST。

```bash
# ❌ 回 {"status": "success"}，但讀回來是空的
curl -X PUT ".../phase1-interface/NAME" \
     -d '{"internal-domain-list":[{"domain-name":"example.com"}]}'

# ✅ 正確
curl -X POST ".../phase1-interface/NAME/internal-domain-list" \
     -d '{"domain-name":"example.com"}'
```

怎麼判斷是不是 member_table——查 schema：

```bash
curl -sk "$FGT/api/v2/cmdb/vpn.ipsec/phase1-interface?action=schema" -H "Authorization: Bearer $TOKEN"
```

看該欄位有沒有 `"member_table": true`。

**結論很簡單但很重要：改完一定要讀回來確認，不能信 API 回的 `success`。**

---

## 幾個查問題的順手工具

**診斷 VPN，第一個該看的是 `event/vpn` log 的 `reason` 欄位**，比翻 traffic log 有效太多：

```bash
curl -sk "$FGT/api/v2/log/memory/event/vpn?count=400" -H "Authorization: Bearer $TOKEN"
```

搭配 `cookies` 判斷卡在哪一階段：

- **responder cookie 全 0** → 第一個封包就被打回（純粹加密提案不合）
- **雙方 cookie 都非零** → 已進到認證階段，問題在帳號/憑證那側

**兩個會誤導你的陷阱：**

`/api/v2/log/memory/traffic/local` **回傳的其實是 `subtype: forward` 的資料**，不是真正的 local-in 流量。我一度用它來「證明 IKE 封包沒到達」，差點做出完全錯誤的判斷——打到 FortiGate 自己的 IKE 屬於 local-in，那個端點看不到。要看真相請用 `event/vpn` 或 CLI `diagnose debug application ike -1`。

**NAT hairpin 會讓內網測試失真**。在自家 WiFi 上連自家公網 IP，FortiGate 看到的來源是內網位址、客戶端以為在連外網，NAT-T 的位址雜湊對不起來，協商會走偏。**內網測 VPN 失敗不代表真的壞**——而且手機已經在家裡網路上了，本來也不需要 VPN。

---

## 方法論：這次真正學到的東西

技術細節之外，這次過程有兩件事值得記下來。

**第一，「查了一個地方沒有」不等於「不存在」。**

DNS 那題我一度下了明確結論：「IKEv2 沒有推送 DNS 網域的機制，只能全隧道或改用 IP 存取」。理由是我查了 `domain` 欄位，發現它是 Cisco Unity（IKEv1 專用）的東西。

結論下得很果斷，但**是錯的**。phase1-interface 有 200 個欄位，我只查了一個就放棄。後來被質疑「應該要有更多針對 DNS 的設定才對」，回頭把整份 schema 窮舉一遍，才找到 `internal-domain-list`。

```bash
# 與其猜欄位名稱，不如直接列出全部
curl -sk "$FGT/api/v2/cmdb/vpn.ipsec/phase1-interface?action=schema" -H "Authorization: Bearer $TOKEN" \
  | python3 -c "
import json,sys
ch = json.load(sys.stdin)['results']['children']
for k,v in sorted(ch.items()):
    if any(w in k.lower() for w in ('dns','domain','search')):
        print(f\"{k:26} {v.get('help','')[:60]}\")
"
```

Schema 的 `help` 欄位是最可靠的來源——比記憶、比部落格、比我自己的直覺都可靠。

**第二，症狀相同不代表原因相同。**

這次「隧道通但流量是 0」出現了四次，四次的根因分別在：認證層、政策層、DNS 服務層、客戶端作業系統行為。如果第一次修好之後就假設「同樣的症狀＝同樣的問題」，後面三個永遠找不到。

每一次都要重新問「這一次的證據指向哪裡」——`in/out` 的數字組合、log 裡有沒有 deny、DNS 伺服器有沒有收到查詢，這些都是不同層的指紋。

---

## 檢查清單

### 建 dial-up IPSec VPN 時

- [ ] phase1/phase2 加密參數（**客戶端能力才是限制**，不是伺服器）
- [ ] **隧道介面 IP** — 手動建的一定要補，Wizard 建的才會自動配
- [ ] mode-cfg IP 池**不與其他隧道重疊**
- [ ] **政策的來源位址物件 = IP 池範圍**（改池必同步改物件）
- [ ] 使用者群組欄位（IKEv1 XAuth 和 IKEv2 EAP 都用 `authusrgrp`）
- [ ] IKEv2 用 EAP：`eap enable` + **`eap-identity send-request`**
- [ ] DNS：`ipv4-dns-server1` + **`internal-domain-list`**（split tunnel 必要）
- [ ] 內部 DNS 的 ACL / view 要涵蓋 VPN 池
- [ ] **每項改完讀回來確認**（API 可能靜默失敗）

### 判斷「隧道通但沒流量」

1. `in=0 out=0` 雙向全 0 → 政策層擋掉，或客戶端根本沒送
2. traffic log 連 deny 都沒有 → 封包沒進防火牆
3. 有 out 沒 in → 路由 / 回程問題
4. split tunnel 時，**先確認測試的目標在 split-include 範圍內**——測不在範圍內的目標，0 流量是正常的

---

## 最後

SSL VPN 在各家廠商都在被淘汰（Fortinet 拿掉 tunnel mode、其他家陸續有 CVE 和棄用公告）。如果你也在做這個遷移，我的建議是：

**不要一次切換。** 新舊並存過渡，讓每種客戶端都實際測過再收掉舊的。IPSec 的相容性問題幾乎全部在客戶端側，而你沒辦法在防火牆上驗證別人手機支援什麼。
