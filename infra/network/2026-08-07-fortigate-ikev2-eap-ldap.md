<!-- ═══════════════════════════════════════════════════════════════════
     🛑 發布前確認

     本文已去識別化：網域 → example.com / corp.example
     IP → RFC5737 文件用位址、RFC1918 內網位址
     群組名 → IT_Admin / VPN_Users，帳號 → alice
     所有 log 皆為真實輸出，僅置換識別資訊，行號與訊息未改。

     與另一篇〈FortiOS 7.6 拿掉了 SSL VPN…四種死法〉是同一次遷移的產物，
     但主題獨立，可各自發布。若兩篇都發，建議在本文開頭互相連結。

     ── 2026-08-07 晚間修訂（已發布後更正）──
     初版寫「IKEv1 dial-up 用 PSK 必須跑 aggressive mode」，該說法【錯誤】，
     當天稍晚以三平台實測推翻（main mode 可行且不外洩 PSK 雜湊）。
     受影響段落已改寫，「四個選項」表中 IKEv1 那列的代價欄一併更正。
     若初版已被轉載，該處需一併勘誤。
     ═══════════════════════════════════════════════════════════════════ -->

# 同一個 AD 群組，IKEv1 連得上、IKEv2 連不上 —— FortiGate 一個沒有寫在文件裡的硬限制

寫於 2026-08-07（同日晚間修訂，見文末「修訂紀錄」）

---

如果你正在把 FortiGate 的 SSL VPN 遷到 IPSec，而且使用者是 AD / LDAP 帳號，這篇可能會省下你好幾天。

結論先講：

> **FortiGate 的 IKEv2 + EAP 無法直接對 LDAP/AD 認證。**
> 這不是設定錯誤、不是版本 bug、不是客戶端問題——是**認證協定與後端的結構性不相容**。
> 而且**沒有任何一個錯誤訊息會告訴你這件事**。

---

## 症狀

一條設定完全正常的 IKEv2 dial-up 隧道，`authusrgrp` 指向一個 AD 群組。使用者連線後卡住，最後失敗。

`diagnose debug application ike -1` 看起來**一路順暢到最後一步**：

```
SA proposal chosen, matched gateway ipsec-ike2
gw validation OK                              ← 身分驗證過了
FCT EAP 2FA extension vendor ID received      ← 客戶端能力也談成了
responder preparing EAP identity request      ← 開始要帳號
EAP user "alice"                              ← 帳號讀到了
auth group IT_Admin                           ← 群組也讀到了
EAP ... pending
    ↓
detected retransmit
waiting for RADIUS response                   ← 卡在這，無限重送
```

**隧道建起來了、PSK 對了、身分驗證過了、EAP 交握完成了、帳號和群組都正確讀取了。**

然後就停在那裡。

### 而這個錯誤訊息會把你帶往完全錯誤的方向

`waiting for RADIUS response` 看起來像什麼？

- RADIUS 伺服器掛了
- RADIUS 太慢
- 網路不通
- 2FA 在等使用者輸入

**全部都不是。** 而且如果你的架構裡根本沒有 RADIUS（像我一樣，後端是 LDAP），這行訊息本身就已經在誤導你了——它是 IKE daemon 的通用訊息，不管後端是 RADIUS、LDAP 還是 TACACS+，等待認證回應時都印這句。

---

## 決定性的一個測試

我當時提出了兩個假設，都錯了。推翻它們的不是更多 log，是使用者一句話：

> **「但這個群組在 IKEv1 是能用的。」**

於是做了一個把變數壓到最小的測試：

| 固定不變 | 變數 |
|---|---|
| 同一個 AD 群組 `IT_Admin` | **只有 IKE 版本** |
| 同一個帳號 `alice` | |
| 同一台 FortiGate、同一個 AD 後端 | |
| 同一台筆電、同一個 FortiClient | |

結果：

```
IKEv1 + XAuth  →  ✅ 通
IKEv2 + EAP    →  ❌ 卡住
```

**一個變數，兩種結果。** 這直接排除掉所有「設定錯了」「群組沒設好」「使用者物件不存在」的假設——因為那些東西兩邊完全共用。

---

## 真相在 fnbamd

`diagnose debug application fnbamd -1`（fnbamd 是 FortiGate 的認證代理）：

```
[1774] Rcvd auth req ... for alice in IT_Admin  prot=7 svc=9
[376]  verify_local_user_match_and_update-Found a matching user in CMDB 'alice'
[788]  Loaded LDAP server 'CORP_LDAP' for user 'alice' in group 'IT_Admin'
[856]  Total LDAP servers to try: 1                              ← LDAP 有被載入
[1795] fnbamd_ldap_auth_ctx_init-User: alice, password query: 1  ← 準備用密碼做 bind
[1032] __ldap_auth_ctx_prep-Invalid credential                   ← ⚠️ 拿不到密碼
[438]  ldap_start-Failed to init ldap ctx for CORP_LDAP          ← 放棄 LDAP
[316]  radius_start-eap_local=1
[906]  Loaded RADIUS server 'EAP_PROXY' → 127.0.0.1:1812         ← 退回內建 EAP 代理
[965]  Sent radius req ... EAP msg from client: 02 4C 00 09 01 61 6C 69 63 65
                                                          ^^^^^^^^^^^^^^^^^^ "alice"
       （之後逾時無回應 → 就是 ike log 看到的 waiting for RADIUS response）
```

注意這幾件事：

- **使用者物件存在**（`Found a matching user in CMDB`）
- **LDAP 伺服器有被載入**（`Total LDAP servers to try: 1`）
- **失敗發生在「準備憑證」這一步**（`Invalid credential`），不是連線失敗、不是帳號不存在
- 送給 `EAP_PROXY` 的封包裡**只有身分字串 `alice`，沒有任何密碼材料**

---

## 機制：LDAP 要明文，EAP 沒有明文

**LDAP 認證的本質是「拿帳號 + 密碼去做 bind」。** 那需要明文密碼。

```
IKEv1 → XAuth → 客戶端送【明文密碼】→ 拿去對 LDAP bind → ✅

IKEv2 → EAP-MSCHAPv2 → 先送 identity，密碼靠挑戰/回應驗證
                       → 全程只有雜湊，從不產生明文 → bind 不了 → ❌
```

FortiGate 拿不到能 bind 的憑證，於是放棄 LDAP，退回內建的 `EAP_PROXY`。

但 `EAP_PROXY` 是設計來對**本機帳號**做 MSCHAPv2 的——密碼存在 FortiGate 上，所以它算得出對應的雜湊來比對。`alice` 是 LDAP 使用者，**FortiGate 手上根本沒有她的密碼**，無從比對，於是沒有回應，於是逾時。

一切都合理，而且**每一步都沒有錯誤訊息**。

### 這解釋了所有現象

| 現象 | 解釋 |
|---|---|
| 同一個群組，IKEv1 能用 | XAuth 送明文密碼 → LDAP bind 成功 |
| IKEv2 不能用 | EAP 只有雜湊 → `Invalid credential` |
| **本機帳號**走 IKEv2 可以 | 密碼存在 FortiGate → `EAP_PROXY` 算得出雜湊 |
| 群組設定看起來完全正常 | 因為它確實正常——問題不在群組 |
| 訊息說 `waiting for RADIUS response` | 誤導——LDAP 在更早的地方就被放棄了 |

---

## 為什麼這對遷移是大事

多數企業的 SSL VPN 是這樣的：**使用者在 AD，FortiGate 上有一堆對應的 LDAP 群組，policy 依群組給權限。**

遷到 IPSec 時，很自然會想：「IKEv2 比較新、比較安全，直接用 IKEv2 吧，群組設定照搬。」

**那條路走不通。** 而且你會在完全錯誤的地方找原因——因為錯誤訊息指向 RADIUS，而你的架構裡可能連 RADIUS 都沒有。

### 四個選項

| 方案 | 說明 | 代價 |
|---|---|---|
| **RADIUS 擋在 AD 前面** | FortiAuthenticator / Windows NPS / freeradius + ntlm_auth。它們能用 MSCHAPv2 對 AD 驗證，FortiGate 對 RADIUS 做 EAP passthrough | **唯一能保留 AD 集中管理的路**，但要多一套系統 |
| **憑證認證**（`authmethod signature`） | 完全不用密碼 | 要建內部 CA、簽發並派送用戶端憑證 |
| **全改本機帳號** | 密碼存在 FortiGate，EAP 就能用 | 失去集中管理，幾十個群組不現實 |
| **留在 IKEv1 + XAuth** | 現在就能用，不用改任何後端 | **用 main mode 的話幾乎沒有代價**（見下節）；唯一已知限制是 iOS 的 2FA 提示框 |

---

## 一個好消息：遷移其實是兩個獨立的決定

我原本把它想成一件事，實際上是兩件：

```
決定一：SSL VPN → IPSec      ← 現在就能做（IKEv1 + 既有 AD 群組，零改動）
決定二：IKEv1 → IKEv2        ← 要等 RADIUS，可以慢慢來
```

**第一件的阻力比想像小得多**——AD 不用動、群組不用動、使用者帳號不用動。

而且 **FortiGate 同一個介面上可以並存多條隧道**，靠「IKE 版本 + 交換模式」分辨：

```
tunnel-a   IKEv1 main mode
tunnel-b   IKEv1 aggressive mode
tunnel-c   IKEv2
           ↑ 同一個 WAN 介面，三條同時活著
```

所以第二件事可以**一批一批搬**，IKEv1 全程當退路，出事隨時退回。不需要挑一個晚上全公司一起賭。

### 如果決定先用 IKEv1，請用 main mode——而且有三個欄位缺一不可

很多資料（包括我自己一開始的認知）會告訴你：

> ❌「IKEv1 dial-up 用 PSK 必須跑 aggressive mode，因為 gateway 得先知道對方身分才能挑對 PSK。」

**這是錯的，我實測推翻了它。**

真正的限制是：**當同一個介面上有多條 dial-up 隧道、各自不同 PSK 時**，main mode 才選不出該用哪一把。如果那個介面上**只有一條 dial-up、只有一把 PSK**，gateway 根本不需要選——直接用就好。

這件事很要緊，因為 **main mode 的身分交換是加密的，不會把 PSK 雜湊送上線**。也就是說「用 IKEv1 過渡」原本被認為要付的安全代價，**其實不存在**。

實測結果（同一台 FortiGate、同一個 AD 群組、同一組帳號）：

```
IKEv1 + main mode + PSK + XAuth
   iPhone FortiClient    ✅
   Android FortiClient   ✅
   macOS FortiClient     ✅
```

不需要 RADIUS、不用動 AD、PSK 雜湊不外洩。

#### 三個欄位，缺任何一個都會表現成「某個平台不能用」

| 欄位 | 沒設時的症狀 | 而錯誤訊息會說 |
|---|---|---|
| `set mode main` | **iOS 連協商都進不去**——它只送 main mode，aggressive 的隧道在協定層就服務不了它 | `no SA proposal chosen`（看起來像演算法談不攏）|
| `set dhgrp 2 5 14`<br>（**不能只給一個**） | **Android** 在 phase 2 反覆失敗，靠重試才偶爾成功（成功也要 5 秒，正常 0.4 秒）| `peer SA proposal not match`（不告訴你是哪個參數）|
| `set domain "example.com"` | **DNS 一筆都不送進隧道**——不是解析失敗，是根本沒送 | **完全沒有訊息，靜默失敗** |

幾個補充，每個都是實測踩出來的：

- **`domain` 是 IKEv1 專用的 Cisco Unity 欄位**（IKEv2 用 `internal-domain-list`）。沒有搜尋網域時，**iOS/macOS 只把「符合指定網域」的查詢送進隧道 → 沒有網域可比對 = 零筆**；Android 等客戶端則是全部送。結果就是「一支手機不行、其他都可以」——**其實是同一個欄位**，跟手機新舊、跟作業系統都無關。
- **`dhgrp` 上限 3 組**。給 4 組會回 `too many argument to set 4, 3`——訊息看起來像參數無效，實際是數量超限。
- **aggressive mode 的 DH group 在第一個封包就固定了**，客戶端沒得協商。所以如果你要留一條 aggressive 給特定客戶端，proposal 一定要開寬，否則等於把它擋在門外。
- 別忘了把 **VPN IP 池加進內部 DNS 的 ACL**（BIND 的 `acl internal`），否則 VPN 使用者會落到 external view，拿到公網記錄而不是內網 IP。

#### 至於 PSK 強度

即使用 main mode 不外洩雜湊，**PSK 夠長夠亂仍然值得做**——成本幾乎為零。如果你因為相容性必須保留一條 aggressive 隧道給某些客戶端，那條就真的會在認證前送出 PSK 雜湊，這時 PSK 強度就是唯一的防線。

#### 一個沒被解決的限制：iOS + 2FA

實測下來，**iOS FortiClient 不會渲染 XAuth 的第二輪挑戰**——FortiGate 有正常寄出 OTP，但手機不跳輸入框，連線就卡在那裡。同一套設定下 Android 跳得出來、可以用。

這跟前面 IKEv2/EAP 缺 Fortinet 私有擴充是同一家族的客戶端限制。**推論**（未實測）：需要使用者輸入 6 碼的 2FA（email / SMS / TOTP）在渲染不出提示框的客戶端上是結構性不可能的；理論上只有「推播核准」這種不經過 VPN 客戶端的方式能繞過。

---

## 怎麼在自己的環境驗證

兩個指令，五分鐘：

```
# 1. 看 IKE 層（會看到 waiting for RADIUS response，但那是誤導）
diagnose debug reset
diagnose debug application ike -1
diagnose debug enable
    ← 用 IKEv2 連一次

# 2. 看認證層（真相在這）
diagnose debug reset
diagnose debug application fnbamd -1
diagnose debug enable
    ← 再連一次

diagnose debug disable
diagnose debug reset
```

**要找的關鍵字：**

- `__ldap_auth_ctx_prep-Invalid credential` → 就是本文講的問題
- `ldap_start-Failed to init ldap ctx` → 同上
- `Loaded RADIUS server 'EAP_PROXY'` → 它退回內建代理了

**對照組**：同一份設定切回 IKEv1 再測一次。如果 IKEv1 通、IKEv2 不通，就確定了。

---

## 順帶一提：這個坑的形狀

這次除錯裡，**每一個失敗的錯誤訊息都指向錯誤的方向**：

| 訊息 | 你會以為 | 實際上 |
|---|---|---|
| `waiting for RADIUS response` | RADIUS 掛了 / 太慢 | LDAP 在更早的地方被放棄 |
| `no SA proposal chosen` | 加密演算法談不攏 | 交換模式不同（main vs aggressive） |
| `gateway validation failed` | PSK 錯 / 憑證問題 | 客戶端送的身分識別**型別**不同 |
| `peer SA proposal not match` | 演算法沒開夠 | DH group 只開了一個，客戶端挑了別的 |
| `too many argument to set 4, 3` | 參數值無效 | 只是**數量**超過上限（`dhgrp` 最多 3 組）|
| **（完全沒有訊息）** | 設定應該生效了吧 | `domain` 沒設 → DNS 查詢一筆都沒送進隧道 |

六個訊息，六次誤導。共同點是：**它們都描述「在協定的哪一層失敗」，而不是「哪個設定欄位錯」**，而那兩者在 IPSec 裡經常隔著三四層。

最後那一列尤其值得記著——**最貴的坑是沒有錯誤訊息的那個**。隧道建起來了、狀態顯示正常、使用者卻說「連得上但什麼都打不開」，而你翻遍 log 找不到任何一行紅字。

所以遇到 IPSec 問題，我現在的第一動作不是改設定，是**多開一層 debug**——`ike` 之外再看 `fnbamd`。上面那層的訊息通常只是下面那層的回音。

---

## 修訂紀錄

**2026-08-07 晚間（發布當日更正）**

初版寫著：

> ❌「IKEv1 dial-up 用 PSK **必須**跑 aggressive mode（gateway 得先知道對方身分才能挑對 PSK，main mode 的訊息順序做不到），而 aggressive mode 會在認證前把 PSK 雜湊送上線。」

**這個說法是錯的。** 當天稍晚我在同一台設備上實測推翻：建一條 IKEv1 **main mode** 的 dial-up 隧道，iOS / Android / macOS 三個平台全部連得上。

錯在哪：我把「同一介面上多條 dial-up、各自不同 PSK」這個**特例**當成了通則。單一 dial-up 隧道時 gateway 不需要選 PSK，main mode 完全可行——**而且身分交換是加密的，不外洩雜湊。**

這個錯誤不是無關痛癢的細節：它撐著原文「先用 IKEv1 過渡要付安全代價」的整段結論，而那個代價**其實不存在**。受影響的段落已重寫，「四個選項」表中 IKEv1 那列也一併更正。

順帶一提，這正好呼應本文的主題——**IPSec 裡最容易出錯的不是設定，是你以為自己知道的事**。我花了大半天才把一個自己重複講了三次的「常識」推翻，靠的不是想通，是把 `diagnose debug application ike -1` 打開來看。
