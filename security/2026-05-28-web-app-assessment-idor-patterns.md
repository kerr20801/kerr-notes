# Multi-Tenant SaaS 最常見的致命錯誤：IDOR 與公司隔離失效

寫於 2026-05-28

---

剛完成一個 B2B SaaS 平台的 Web Application Security Assessment。

4 個 Critical 漏洞，全部源自同一個根本原因。

---

## 問題在哪

Multi-tenant SaaS 的核心假設是：
A 公司的資料，B 公司看不到。

但很多系統的實作是：

- 驗證你有沒有登入 ✅
- 驗證你的角色夠不夠 ✅
- 驗證這筆資料是不是你公司的 ❌

最後這步沒做，就是 IDOR。

---

## 實際長什麼樣

Admin 呼叫一個標準的列表 API：

```
GET /v1/licenses/list
```

回傳的不只是自己公司的授權，
是系統裡所有公司的授權，
還附帶明文的 api_key 跟 api_secret。

一個 request，全洩漏。

---

## 為什麼這麼常見

開發流程的問題。

功能開發時只測「能不能用」，
沒有測「能不能拿到不該拿的東西」。

RBAC 做了，但 RBAC 只管角色，
不管資料所有權。

兩件事，很多人以為做了一個就夠了。

---

## 修法其實很簡單

```python
# 每個涉及資源存取的地方加這一行
if resource.company_id != current_user.company_id:
    raise HTTPException(403)
```

一行，同時關掉四個 Critical。

難的不是修，是**知道要加**。

---

## AI 怎麼加速這個過程

傳統驗證 IDOR 要手動換 token、換 ID、逐一測試。

AI 輔助的做法：
- 自動枚舉所有端點
- 批次替換身份 token
- 比對回應差異，標記異常

人工只需要確認：這個異常是真的漏洞，還是預期行為。

驗證速度快 10 倍，但判斷還是人在做。

---

*所有案例均已匿名化處理*
