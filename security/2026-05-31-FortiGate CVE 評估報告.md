# 🔑 FortiGate API Token 深度審計報告 — 2026-05-31

> **目標：** [[PRIVATE_IP_76836b]]（FortiGate 60E）
> **認證方式：** API Token（Bearer）
> **執行時間：** 2026-05-31 上午
> **重要：** 此報告包含已授權測試取得的設定資訊，請妥善保管

---

## 設備基本資訊

| 項目 | 值 |
|------|-----|
| 型號 | FortiGate 60E (FGT60E) |
| 版本 | **v7.4.11 build 2878**（精確版本，確認所有已知 CVE 已修）|
| Hostname | Lab_FW |
| Serial | [[PRIVATE_IP_76836b]] |
| 時區 | Asia/Taipei |
| 管理 Port | 443 |
| 管理 VDOM | root |

---

## 🗺️ 完整網路拓樸（原本未知）

| 介面 | IP | 段 | 用途 |
|------|----|----|------|
| wan1 | **123.123.123.123** | 公網 | 主線（中華電信）|
| wan2 | **55.55.55.55** | 公網 | 備線（中華電信）|
| Kerr_Lan | [[PRIVATE_IP_76836b]]/24 | 內網 LAN | 伺服器區 |
| Kerr_Wifi_Lan | [[PRIVATE_IP_76bbc6]]/24 | WiFi | 辦公室 WiFi |
| dmz | [[PRIVATE_IP_8fd78a]]/24 | **DMZ** | ⚠️ 之前完全不知道 |
| Advpn | [[PRIVATE_IP_9ec44d]] | VPN tunnel | ADVPN Hub |
| fortilink | [[PRIVATE_IP_039bc2]]/24 | Fortilink | FortiSwitch 管理 |

### 對外 VIP（公網暴露）

| 名稱 | 公網 | 映射到 | Port |
|------|------|--------|------|
| vip_nginx_proxy | 123.123.123.123:443 | **[[PRIVATE_IP_a63419]]:443** | HTTPS |

> 只有 hackmd.abc.com (.45) 對外，其他服務均在 NAT 後。

---

## 🔴 DMZ 新發現：[[PRIVATE_IP_df7489]]

### 服務
- **Port 22：** SSH
- **Port 8080：** `Werkzeug/3.1.8 Python/3.12.3` → **台股 AI Dashboard**

### 關鍵特徵
- HTTP Header：`X-Cache: MISS from ai` + `Via: 1.1 ai (squid/5.9)`
  → DMZ 裡有一個 **Squid proxy** 叫 `ai`
- Dashboard 頁面：`<title>台股AI Dashboard</title>`（Stock AI 系統）
- 防火牆規則：
  - `Stock_ai_To_Server_hackmd`：DMZ → Server_Lan HTTPS（存取 hacmkd.abc.com）
  - `Stock_ai_To_Server_Gitlab`：DMZ → Server_Lan SSH（存取 GitLab）
  - `Server_Lan_To_Stock`：Server_Lan → DMZ ALL（伺服器可完整存取 DMZ）

### 安全建議
- Stock AI 的 Squid proxy 暴露了內網代理路徑
- DMZ → GitLab SSH 規則：若 DMZ 被攻破可直接跳進 （GitLab）

---

## 🔐 SSL VPN 設定審計

| 項目 | 值 | 評估 |
|------|-----|------|
| 狀態 | enable | — |
| Port | **16080**（非標準）| ✅ 不用 443，略微降低掃描暴露 |
| 客戶端憑證 | disable | ⚠️ 只靠帳密，無憑證雙因子 |
| Auth timeout | 28800 秒（8小時）| ⚠️ Session 太長 |
| 登入嘗試上限 | **2 次** | ✅ 很嚴格 |
| 封鎖時間 | **86400 秒（24小時）**| ✅ 超嚴格 |

---

## 👥 SSL VPN 使用者清單

| 帳號 | 狀態 | 2FA | 風險 |
|------|------|-----|------|
| user1 | enable | ❌ disable | ⚠️ 管理員沒開 2FA |
| user2 | enable | ❌ disable | ⚠️ |
| user3 | enable | ❌ disable | ⚠️ |
| guest | enable | ❌ disable | 🔴 **guest 帳號！** |

> 🔴 **`guest` 帳號存在且啟用**，無 2FA。需確認密碼強度和 portal 存取範圍。
> ⚠️ **4 個帳號全部沒有開 2FA**，SSL VPN 只靠帳密保護。

---

## 🔒 密碼政策審計

| 項目 | 值 | 評估 |
|------|-----|------|
| 密碼政策 | **disable** | 🔴 完全沒有密碼政策！ |
| 最短長度 | 無要求 | 🔴 可以設空密碼 |
| 複雜度要求 | 無 | 🔴 |
| 密碼到期 | disable | ⚠️ 永不過期 |

---

## 🌐 ADVPN 隧道（Site-to-Site VPN）

| Tunnel | Remote GW | PTR | 狀態 | 流量（進/出）|
|--------|-----------|-----|------|-------------|
| Advpn_1 | 1.3.2.9 | hinet-ip.hinet.net | **UP** | 25.6 MB / 12.2 MB |
| Advpn_2 | 6.8.88.17 | dynamic-ip.hinet.net | **UP** | 1.46 MB / 0.79 MB |
| Advpn_3 | 1.63.17.3 | dynamic-ip.hinet.net | UP（低流量）| 185 KB / 150 KB |

> 有 3 個 ADVPN site 連線，代表有其他辦公室或遠端站點。Advpn_1 流量最大，是主要站點。

---

## 🛡️ 管理介面安全設定

| 項目 | 值 | 評估 |
|------|-----|------|
| HTTPS SSL 版本 | TLS 1.2 + TLS 1.3 | ✅ |
| Admin 鎖定閾值 | 3 次失敗 | ✅ |
| Admin 鎖定時間 | 60 秒 | ⚠️ 稍短，建議 300 秒以上 |
| 登入前 Banner | disable | Low（建議啟用法律警告）|
| 管理帳號（API 可見）| 0（token 無讀取 admin 權限）| — |
| 目前登入管理員 | kerr（super_admin）from [[PRIVATE_IP_f3efd5]] | 正常 |

---

## 防火牆政策摘要（共 20 條）

| 重要規則 | 方向 | 說明 |
|----------|------|------|
| LAP_TAP_AUTO_BLOCK | any → any DENY | LAP AI 自動封鎖規則（Policy 1，最優先）|
| Office_Lan_To_Wan | LAN → WAN | 辦公室對外 |
| LAP _TO_WAN | LAN → WAN | LAP AI 對外（注意名稱有空格）|
| Stock_ai_To_Server_Gitlab | DMZ → LAN SSH | ⚠️ DMZ 可 SSH 進 GitLab |
| Stock_ai_To_Server_hackmd | DMZ → LAN HTTPS | DMZ 存取 Outline |
| Wan_To_Lan_Nginx_Proxy_45 | WAN → LAN | 公網 HTTPS 進 .45 |
| SSL_Vpn_To_Lan | ssl.root → LAN | SSL VPN 存取內網 |
| SSL_Vpn_To_Lan_user2 | ssl.root → Advpn | user2 的 VPN 只能進 ADVPN |

---

## 風險摘要

| 嚴重度 | 項目 | 建議 |
|--------|------|------|
| 🔴 HIGH | `guest` SSL VPN 帳號啟用 | 停用或設強密碼 + 2FA |
| 🔴 HIGH | 密碼政策完全停用 | 啟用，最短 12 字元 + 複雜度 |
| ⚠️ MEDIUM | 所有 VPN 用戶無 2FA | 啟用 FortiToken 或 SMS OTP |
| ⚠️ MEDIUM | DMZ → GitLab SSH 規則 | 確認 Stock AI 是否真的需要 Git push |
| ⚠️ MEDIUM | Admin 鎖定時間只有 60 秒 | 建議拉長到 300-600 秒 |
| ⚠️ MEDIUM | Auth timeout 8 小時 | 建議縮短到 4 小時 |
| ℹ️ LOW | 無 pre-login banner | 建議加法律警告聲明 |

---


