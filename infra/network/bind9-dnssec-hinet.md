# BIND9 + DNSSEC：為什麼內部 DNS 解不了外部域名

> **環境**：Ubuntu 24.04，BIND9，上游 DNS 使用中華電信 168.95.1.1

---

## 症狀

內部 DNS server 裝好 BIND9 之後，內網機器解自己的 zone 沒問題，但解外部域名（`.com`、`.tw`、`.net`）全部回 `SERVFAIL`：

```bash
$ nslookup google.com 172.16.x.x
Server: 172.16.x.x
Address: 172.16.x.x#53

** server can't find google.com: SERVFAIL
```

換成 8.8.8.8 就正常。

---

## 根本原因

BIND9 預設 `dnssec-validation auto`。

這個設定的意思是：BIND 會驗證上游回傳的 DNS 答案有沒有帶 DNSSEC 簽章。

問題：**中華電信 168.95.1.1 回應不帶 DNSSEC 簽章**。

BIND 收到沒有簽章的答案，驗證失敗，直接丟掉，對 client 回 `SERVFAIL`。

---

## 為什麼用 8.8.8.8 就好

Google DNS 的回應帶有完整的 DNSSEC 鏈，BIND 驗得過，所以沒問題。

---

## 解法

`/etc/bind/named.conf.options` 改一行：

```
options {
    // ...
    dnssec-validation no;   // 改這行，預設是 auto
    // ...
};
```

然後重啟：

```bash
systemctl restart named
```

---

## 另一個常見原因：系統時鐘偏差

DNSSEC 簽章有時效性。如果 DNS server 的系統時間跟現實差超過幾分鐘，BIND 會認為簽章已過期，同樣回 `SERVFAIL`。

解法：裝 chrony 並確認時間同步正常。

```bash
apt install chrony -y
systemctl enable --now chrony
chronyc tracking  # 確認 offset 在合理範圍
```

**完整排查順序：**

1. `dig @localhost google.com` — 確認是 SERVFAIL 還是 NXDOMAIN
2. `dig @8.8.8.8 google.com` — 換上游測試，如果正常就是 DNSSEC 問題
3. `date` 確認系統時間正確
4. `journalctl -u named | grep -i dnssec` — 看 BIND log 有沒有 validation error

---

## 補充：為什麼不直接用 8.8.8.8 當上游

如果你的內網環境有自己的 zone（`kerr.internal`、`*.lab` 之類），必須有內部 DNS 來解這些名稱。

外部域名通過內部 DNS forward 到上游，是標準架構。問題是上游 ISP DNS 不帶 DNSSEC 簽章這個坑，文件沒人提。

---

*記錄於 2026-05-13，Ubuntu 24.04 + BIND9 重建 DNS server 實際踩坑*
