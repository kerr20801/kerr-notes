<!-- ═══════════════════════════════════════════════════════════════════
     🛑 發布前必讀 —— 2026-08-05 更新，本文有一段已知錯誤，先修再發

     〈2FA 能不能用，取決於「客戶端」而不是「IKE 版本」〉那節（約 L231–277，
     以及 L531 檢查清單那條）把分界寫成「原生用戶端 vs FortiClient」——**這是錯的**。

     2026-08-05 晚間受控實測（同一支手機、同一個 FortiClient、同一帳號、同一條隧道，
     只改第二因素）證實：**失敗的那台手機從頭到尾都在跑 FortiClient**。
     真正的分界是 **FortiClient 桌面版 vs 行動版**。

       IKEv2 + 2FA + FortiClient 桌面版(macOS)  ✅  15:04:58 寄 OTP → 15:05:11 連上
       IKEv2 + 2FA + FortiClient 行動版         ❌  eapuser 認證通過後 0.1~0.6s 拆掉
                                                    第二因素事件 0 筆（email / 已綁定 FortiToken 皆同）

     連帶要修的還有：
       • 「怎麼從 log 分辨客戶端：看 fctuid」——**fctuid 不可靠**，phase 1 早期記錄
         本來就不帶這欄位，缺席 ≠ 原生用戶端。這句要刪或改寫。
       • 可補的新料：negotiate-timeout(預設30s) < two-factor-email-expiry(預設60s)
         原廠預設自相矛盾；fortinet-notifications.com 中繼寄 3 次才到。

     完整證據（11 次可重現紀錄 + 毫秒時序）在內部 Outline 該文 Part 4。
     ✅ 2026-08-06 已收斂：MacBook + FortiClient + 2FA 實測成功
        （IKEv2 09:22:40 / IKEv1 09:23:38，兩者皆 tunnel-up）。行動版確定不行。

     ─────────────────────────────────────────────────────────────────
     💡 2026-08-06 另有大量新素材可強化本文，見 Outline 該文 **Part 5**：

     現在文章的主軸是「四種死法」。新素材讓它可以升級成一個更完整的論點：
     **遷移的難處不在設定，而在於一個沒有共識的客戶端生態，
       而且每個失敗的錯誤訊息都指向錯的方向。**

       症狀                              你會以為        真因
       no SA proposal chosen(cookie全0)  演算法不合      交換模式不同(iOS只送main)
       gateway validation failed         PSK錯/憑證      身分識別型別不同(安卓送KEY_ID)
       SA deleted 無原因/0.2s/OTP 0筆     逾時/帳密       行動版無2FA挑戰擴充

     三個根因是「交換模式／身分識別型別／私有擴充有無」——IPsec 規格全都讓
     實作者自選且無預設共識。SSL VPN 好用正是因為它沒有這些選項(走HTTPS、
     TLS交握全球統一)。**IPsec 的彈性就是它的詛咒。**

     還有一段方法論很適合收尾：本次盲試三輪(加演算法、關PFS、差點再加DH group)
     全部無效，兩分鐘的 ike debug 直接給出答案 →
     **改了參數而失敗方式一字不變 = 問題不在那個參數。**
     ═══════════════════════════════════════════════════════════════════ -->

# FortiOS 7.6 拿掉了 SSL VPN，我被迫遷到 IPSec —— 挖出「隧道通了但什麼都不通」的四種死法

寫於 2026-08-04

---

FortiOS 7.6 在低階機型（2GB RAM，60F 這類）**移除了 SSL VPN tunnel mode**。官方立場很明確：改用 IPSec。

我的 60F 從 7.4.12 升到 7.6.7，本來以為是例行升版，結果變成一次完整的 VPN 架構遷移。過程中撞到四個「隧道明明建起來了，但就是不通」的情況——**而且四個的根因完全不同，症狀卻長得一模一樣**。

這篇不講「怎麼設定 IPSec VPN」（官方文件寫得很清楚），只講那些**設定看起來完全正確、但實際上不會動**的地方。

如果你只想看一個重點，**看死法一就好**——那個坑跟 FortiGate 一點關係都沒有，任何用 VPN / overlay 網路又有跑 Docker 的環境都會踩到，而且症狀誤導性極強。

> 文中所有 IP、網域已改為範例值。

---

## 先講升版本身：比預期無聊

| | 升版前 | 升版後 |
|---|---|---|
| 記憶體 | 57~58% | **51%** |
| conserve mode | 從未進入 | 仍為 0 |

**記憶體不升反降**。我原本擔心 2GB 機型跑 7.6 會很緊，結果移除 SSL VPN tunnel 釋出的資源比新版本多吃的還多。設定也全數存活，UTM profile 一個沒掉。

### ⚠️ 一個我連續判斷錯兩次的地方：config 物件 ≠ 功能還在

這段值得完整寫出來，因為它示範了一個很容易犯的錯。

**第一次判斷（錯）**：升版前我以為「7.6 把 SSL VPN 整個拿掉」。

**第二次判斷（也錯，但錯得比較隱蔽）**：升完後我用 API 列出 config 物件的欄位，發現 tunnel 相關欄位全消失、但 `web-mode` 還在，於是下結論「**只有 tunnel mode 被移除，web portal 還在而且可用**」。

```bash
# 我當時用的方法
curl -sk "$FGT/api/v2/cmdb/vpn.ssl.web/portal/full-access" -H "Authorization: Bearer $TOKEN" \
  | python3 -c "import json,sys; print(sorted(json.load(sys.stdin)['results'][0].keys()))"
```

欄位確實是這樣沒錯——但**結論是錯的**。

**實際狀況**（使用者告訴我 CLI 下 `config vpn ssl settings` 是空的，我才回頭驗證）：

| 檢查方式 | 結果 |
|---|---|
| API 讀 `vpn.ssl/settings` | `status: enable`, `port: 16080` ← **看起來還開著** |
| CLI `config vpn ssl settings` | **指令不存在** |
| 實測 port 是否監聽 | **沒有回應**（內網直連測試，排除 NAT 干擾） |

**功能移除後，CMDB 裡的舊設定值還躺在那裡**——API 讀得到，但 CLI 指令已移除、daemon 不啟動、port 不監聽。那是**孤兒設定，不是啟用中的服務**。

**教訓：判斷一個功能死活，要看 runtime 而不是 config。**

- ❌ 只查 config 物件 / API 回傳值
- ✅ 測 port 有沒有在聽、CLI 有沒有這個指令、monitor 端點回什麼

這個坑比「功能還在」更危險——因為它會讓你以為某個攻擊面還存在（或還可用），而做出錯誤的風險判斷。我就差點因此建議使用者去關一個**早就沒在跑的服務**。

---

## 死法一：VPN 網段撞到 Docker，回程封包被吃掉

這個跟 FortiGate 無關，但它是我這次踩到**最通用、也最難查**的一個。

症狀：VPN 連上了，但**只有防火牆本身（gateway）ping 得到，內網其他主機全部不通**。

FortiGate 的 log 長這樣：

```
172.17.100.1 → 10.0.10.21:443   action=timeout   policyid=18   ← GitLab
172.17.100.1 → 10.0.10.80:53    action=accept    policyid=18   ← DNS
```

注意這裡：**`action=timeout` 不是被擋**。它的意思是「政策允許、封包已轉發出去，但沒有任何回應回來，session 逾時了」。如果是政策擋掉會是 `action=deny`。

也就是說：封包**去得了、回不來**。

### 根因

我當初把 VPN 客戶端的 IP 池設在 `172.17.100.0/24`。

**Docker 預設 bridge（`docker0`）用的就是 `172.17.0.0/16`。**

於是任何有跑 Docker 的內網主機，收到來自 `172.17.100.x` 的封包後，查路由表發現「這是 docker0 的網段」，就把回應丟進 Docker 橋接裡——**永遠不會送回防火牆**。

```bash
# 在有 Docker 的內網主機上
$ ip route get 172.17.100.1
172.17.100.1 dev docker0 src 172.17.0.1     ← 回程進了 docker0，不是預設閘道

# 換成安全網段後
$ ip route get 10.99.1.1
10.99.1.1 via 10.0.10.254 dev eth0          ← 正確走預設閘道
```

### 為什麼症狀特別誤導

**通不通取決於那台機器有沒有跑 Docker。**

我環境裡的 DNS 伺服器剛好沒裝 Docker，所以 DNS 查詢**看起來是通的**（log 顯示 `accept`，DNS server 上也真的收到查詢），但同一條 VPN 去連 GitLab（有跑 Docker）就 timeout。

一條 VPN、同一個政策、有些服務通有些不通——這種症狀很容易讓人往「政策設錯」「路由缺了」的方向查，但問題根本不在網路設備上，**而在對端主機的本地路由表被第三方軟體悄悄改過**。

### 哪些網段不能用

Docker 的預設位址池（libnetwork 內建）不只 172.17：

```
172.17.0.0/16
172.18.0.0/16
172.19.0.0/16
172.20.0.0/14     （172.20 ~ 172.23）
172.24.0.0/14     （172.24 ~ 172.27）
172.28.0.0/14     （172.28 ~ 172.31）
192.168.0.0/16
```

**Docker 會從 172.17 一路配到 172.31，外加 192.168.x。** 它是按順序配的——你今天看 `172.19` 沒被用，不代表安全，只代表還沒輪到。等某人 `docker compose up` 一個新專案就會拿走，**而那時候沒人會把「VPN 不通」跟「昨天新開的 container」聯想在一起**。

| 網段 | 安全嗎 |
|---|---|
| `172.16.0.0/16` | ✅ Docker 從 172.17 才開始配 |
| `172.17` ~ `172.31` | ❌ Docker 預設池 |
| `192.168.0.0/16` | ❌ 也在池裡 |
| `10.0.0.0/8` | ✅ Docker 預設完全不碰 |

理論上可以在每台主機的 `/etc/docker/daemon.json` 固定 Docker 的位址池，但那要**每台跑 Docker 的機器都改、都重啟 Docker**（中斷所有 container），新機器還容易漏掉。**直接選一個 Docker 不碰的網段簡單太多**——改 VPN 池只要動一個地方。

---

## 死法二：「Phase 2 不通」其實根本沒壞

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

## 死法三：客戶端相容性才是主戰場

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

### ⚠️ 2FA 能不能用，取決於「客戶端」而不是「IKE 版本」

這條是實測撞出來的，而且**沒有任何錯誤訊息會告訴你原因**。

在一條原生用戶端連得好好的 IKEv2 上，幫使用者開啟 2FA（Email OTP 或 FortiToken）之後——**立刻連不上**。log 長這樣：

```
Negotiate IPsec phase 1  status=success    ← EAP 走了好幾輪
Negotiate IPsec phase 1  status=success
Progress  IPsec phase 1  status=success
IPsec phase 1 SA deleted                   ← 然後就沒了
```

沒有 `reason`、沒有 `xauthuser`、沒有指派 IP，**連 OTP 信都沒寄出去**。

**根因**：標準 EAP-MSCHAPv2 **沒有「認證中途再挑戰一次」的機制**。FortiGate 想送第二因素挑戰時，原生用戶端不知道那是什麼，直接斷掉。

FortiClient 之所以可以，是因為它有**專屬擴充**。在 FortiClient 的連線 log 裡看得到這一行：

```
ike 0:TUNNEL:NNNNN: FCT EAP 2FA extension vendor ID received
```

客戶端主動宣告「我會處理 2FA 挑戰」——原生用戶端不會送這個。

**⚠️ 這裡最容易誤讀成「IKEv2 不能做 2FA」——不是。** 實測矩陣：

| 組合 | 結果 |
|---|---|
| IKEv2 + 2FA + **FortiClient** | ✅ 可以（RADIUS 2FA，連線成功）|
| IKEv2 + 2FA + **原生用戶端** | ❌ 不行 |
| IKEv1 + 2FA + **FortiClient** | ✅ 可以 |

**決定因素是客戶端，不是協定版本。** 兩個 IKE 版本在 FortiClient 下都能做 2FA；原生用戶端則兩者都不行。

> 這點值得特別強調，因為很容易掉進一個推論陷阱：**拿「IKEv1 + FortiClient」跟「IKEv2 + 原生」相比**，看起來像是「IKEv1 能做 2FA、IKEv2 不能」，於是得出「IKEv1 比較安全」的結論。實際上那是**客戶端的差異被誤讀成協定的差異**——而 IKEv1 aggressive 還多背了「PSK 雜湊認證前外洩」這個 IKEv2 沒有的弱點。

**所以真正的二選一是：**

| 你要的 | 代價 |
|---|---|
| 原生用戶端（不裝軟體、手機開箱即用） | **不能有 2FA** |
| 2FA | **所有裝置都要裝 FortiClient**（IKEv1/v2 皆可）|

**怎麼從 log 分辨客戶端**：`event/vpn` 的 `fctuid` 欄位——有值是 FortiClient，`N/A` 是原生用戶端。診斷 2FA 問題時先看這個，免得拿不同客戶端的結果互相比較。

**折衷做法**：同介面並存兩條 IKEv2——一條不帶 2FA 給原生用戶端日常用，另一條綁開了 2FA 的使用者群組給 FortiClient 用，需要碰敏感系統時才走後者。

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

## 死法四：IKEv2 建起來了，但流量是 0

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

## config 物件還在，不代表功能還在

跟上面那個 API 陷阱是一體兩面：**寫入時 API 會騙你（回 success 但沒寫），讀取時 config 也會騙你（欄位還在但功能已死）**。

功能被韌體移除後，CMDB 裡的舊設定往往不會一起清掉。於是：

```bash
# API 說服務開著
$ curl -sk ".../api/v2/cmdb/vpn.ssl/settings" | jq '.results.status, .results.port'
"enable"
16080

# 但 CLI 沒有這個指令了，port 也沒在聽
$ ssh admin@fgt 'config vpn ssl settings'      # → 空的
$ nc -zv fgt 16080                              # → 沒有回應
```

**判斷功能死活的正確方法：**

| 方法 | 可信度 |
|---|---|
| 測 port 有沒有在監聽 | ✅ 最可靠 |
| CLI 有沒有這個指令 | ✅ 可靠 |
| `monitor` 類端點的回應 | 🟡 通常可靠 |
| `cmdb` config 物件的值 | ❌ **可能是孤兒設定** |

這個坑的危險之處在於**方向**：它會讓你以為某個攻擊面還存在，做出過度的風險判斷；或反過來，以為某個防護還在跑，其實早就沒了。

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

**在 macOS 上不要用 `dig` 診斷 VPN 的 DNS。**

這個我自己踩了，還害使用者以為設定壞掉。`dig` 和 `nslookup` **直接讀 `/etc/resolv.conf`**，繞過 macOS 的 mDNSResponder——而 macOS 的「補充解析器」（supplemental resolver，也就是「這個網域用這台 DNS 查」）就活在那一層。

所以 `dig` 永遠只會顯示本地 DNS，即使系統實際上會把 `example.com` 的查詢導進 VPN。**Linux 上的習慣直接套過來就會誤判。**

正確的工具：

```bash
# 看解析器設定（會列出所有補充解析器）
scutil --dns | grep -B2 -A6 "example.com"

# 期待看到：
#   resolver #8
#     domain   : example.com
#     nameserver[0] : 10.0.10.80
#     reach    : 0x00000002 (Reachable)

# 用系統解析器實際查一次（這才是 App/瀏覽器走的路徑）
dscacheutil -q host -a name app.example.com
```

最實際的驗證其實是**直接開瀏覽器**——網頁開得起來就是好的，不管 `dig` 說什麼。

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

- [ ] **IP 池避開 Docker 預設範圍**（`172.17`~`172.31`、`192.168.x`）——用 `172.16.x` 或 `10.x`
- [ ] phase1/phase2 加密參數（**客戶端能力才是限制**，不是伺服器）
- [ ] **隧道介面 IP** — 手動建的一定要補，Wizard 建的才會自動配
- [ ] mode-cfg IP 池**不與其他隧道重疊**
- [ ] **政策的來源位址物件 = IP 池範圍**（改池必同步改物件）
- [ ] 使用者群組欄位（IKEv1 XAuth 和 IKEv2 EAP 都用 `authusrgrp`）
- [ ] IKEv2 用 EAP：`eap enable` + **`eap-identity send-request`**
- [ ] DNS：`ipv4-dns-server1` + **`internal-domain-list`**（split tunnel 必要）
- [ ] **要 2FA 就得用 FortiClient**（原生用戶端不支援第二因素挑戰——**跟 IKE 版本無關**，v1/v2 都一樣）
- [ ] 內部 DNS 的 ACL / view 要涵蓋 VPN 池
- [ ] **每項改完讀回來確認**（API 可能靜默失敗）

### 判斷「隧道通但沒流量」

1. `in=0 out=0` 雙向全 0 → 政策層擋掉，或客戶端根本沒送
2. traffic log 連 deny 都沒有 → 封包沒進防火牆
3. 有 out 沒 in → 路由 / 回程問題
4. split tunnel 時，**先確認測試的目標在 split-include 範圍內**——測不在範圍內的目標，0 流量是正常的
5. **`action=timeout`（不是 deny）+ 只有 gateway 通** → 對端主機的回程路由被搶走，**先查 Docker**：
   在內網主機上跑 `ip route get <VPN客戶端IP>`，看是不是走 `docker0` 而不是預設閘道
6. **有些服務通、有些不通** → 比對那幾台的差異（誰有跑 Docker？），不要先懷疑政策

### 判斷「某個功能還在不在」

- ❌ 查 config 物件（升版移除功能後，舊設定常常留著，API 照樣讀得到）
- ✅ **測 port 有沒有在聽** + **CLI 有沒有這個指令**

---

## 最後

SSL VPN 在各家廠商都在被淘汰（Fortinet 拿掉 tunnel mode、其他家陸續有 CVE 和棄用公告）。如果你也在做這個遷移，我的建議是：

**不要一次切換。** 新舊並存過渡，讓每種客戶端都實際測過再收掉舊的。IPSec 的相容性問題幾乎全部在客戶端側，而你沒辦法在防火牆上驗證別人手機支援什麼。
