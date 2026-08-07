<!-- ═══════════════════════════════════════════════════════════════════
     🛑 發布前確認

     本文已去識別化：網域 → example.com / corp.example
     IP → RFC5737 文件用位址、RFC1918 內網位址
     群組名 → IT_Admin / VPN_Users，帳號 → alice
     所有 log 皆為真實輸出，僅置換識別資訊，行號與訊息未改。

     與另一篇〈FortiOS 7.6 拿掉了 SSL VPN…四種死法〉是同一次遷移的產物，
     但主題獨立，可各自發布。若兩篇都發，建議在本文開頭互相連結。
     ═══════════════════════════════════════════════════════════════════ -->

# 同一個 AD 群組，IKEv1 連得上、IKEv2 連不上 —— FortiGate 一個沒有寫在文件裡的硬限制

寫於 2026-08-07

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
| **留在 IKEv1 + XAuth** | 現在就能用，不用改任何後端 | dial-up + PSK 必須 aggressive mode，PSK 雜湊在認證前上線 |

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

### 如果決定先用 IKEv1，有一件事必須先做

IKEv1 dial-up 用 PSK **必須跑 aggressive mode**（gateway 得先知道對方身分才能挑對 PSK，main mode 的訊息順序做不到），而 **aggressive mode 會在認證前把 PSK 雜湊送上線**——任何人打你的 IKE port 都能抓走，離線慢慢爆破。

**但「可爆破」取決於 PSK 強度。** 一組 40 字元以上的隨機 PSK，離線爆破在現實中做不到。

所以「先用 IKEv1」是站得住的——**前提是那組 PSK 夠長夠亂**。如果現在是人記得住的字串，那才是真的風險。這件事成本幾乎為零，但決定了過渡期安不安全。

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

三個訊息，三次誤導。共同點是：**它們都描述「在哪裡失敗」，而不是「為什麼失敗」**，而那兩者在 IPSec 裡經常隔著好幾層。

所以遇到 IPSec 問題，我現在的第一動作不是改設定，是**多開一層 debug**——`ike` 之外再看 `fnbamd`。上面那層的訊息通常只是下面那層的回音。
