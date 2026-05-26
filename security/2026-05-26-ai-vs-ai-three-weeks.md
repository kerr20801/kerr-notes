# 我用兩個 AI 打架三週，學到了什麼

> 一個防守，一個進攻。兩個 AI，同一個內網，真實基礎設施。

---

## 起點：我以為我的防禦很強

今年初我在家裡 Lab 部署了 NGINX AI Shield——一套我自己寫的 AI 驅動 WAF，三層 ML 模型叠在 NGINX 前面，分析每一個進來的請求。

訓練資料是外部 的真實流量，20,000 多筆攻擊樣本。GradientBoosting、RandomForest、IsolationForest，三個模型投票。Shadow 模式跑了幾週，偵測到一堆東西，我心裡很滿意。

然後我想：**讓另一個 AI 來打它，看看到底多強。**

---

## 龍蝦出場

OpenClaw 是我在另一台隔離 VM 裡跑的 Red Team AI。我叫它龍蝦。

大腦是 Claude Sonnet，工具鏈是 nmap、sqlmap、nuclei（13,060 個 CVE template）、hydra，加上我自己寫的幾個攻擊腳本。它跟 Shield 的關係只有一個原則：**龍蝦只看得到對外介面，不知道 Shield 的任何內部細節。**

5 月 9 日晚上，我給龍蝦一個任務：「打 172.16.32.0/24，找所有安全弱點。」然後我去睡覺。

---

## 第一回合：龍蝦找到真漏洞

早上起來看報告，龍蝦沒有花時間在 NGINX 上。

它做的是 pivot：

```
port_scan(.45) → 全部 401 → 掃整個 /24 子網
→ 發現 .42:8080（FileBrowser，EnableExec: true）
→ 發現 .116:8899（我的基礎設施 AI C.K，無認證）
→ 嘗試 prompt injection 攻擊 C.K
→ 從 C.K 讀到 NGINX 封鎖率、vCenter VM 清單
```

Nuclei 1,673 個 template，12 秒，找到 FileBrowser 登入介面。

Shield 全程偵測到攻擊流量，bypass 從 7,131 暴增到 41,000+。

但當時 Shield 還在 shadow mode，只記錄，不封鎖。

所以龍蝦拿到了兩個真實漏洞：

- **C.K API 8899：無認證（CRITICAL）**
- **FileBrowser EnableExec: true（HIGH）**

Shield 偵測到了，但沒擋住任何東西。

我沉默了一下，然後去修漏洞。

---

## 第二回合：切 Auto Mode

修完之後，我把 Shield 從 shadow 切成 auto——真正封鎖模式。

龍蝦再打一輪。這次結果完全不同：

| 指標 | 第一回合 | 第二回合 |
|------|----------|----------|
| bypass 增加量 | +34,000 | **+261** |
| 被封 session | 0 | **713** |

261 vs 34,000。

龍蝦大部分請求在 NGINX 層就收到 403，根本沒到後端。Shield 把 .28 的所有 session 全部進封鎖清單。

藍隊贏了第二回合。

---

## 第三回合：龍蝦開始學習

龍蝦分析了失敗原因，自己設計了反制工具：

**session_spoof.py** — 把攻擊請求藏在正常流量裡：

```
warmup:   GET /api/status → GET /metrics → GET /health  # 偽裝成監控系統
attack:   [真正的攻擊]
cooldown: GET / → GET /api/v1
UA:       Prometheus/2.30.0
```

**campaign_planner.py** — 把攻擊鏈打散跨 session：

```
傳統攻擊：fingerprint → scan → fuzz → inject  ← L3 一眼認出
Campaign：Session 1 只做 fingerprint，等 3 小時
          Session 2 跳過 port_scan（故意造缺口）
          Session 3 才做 cve_check
```

測試目標換成只有 L1 防守的服務，結果：

**Shield 零記錄。session_spoof 完全有效。**

龍蝦學會了繞。

---

## 然後我發現了更大的問題

5 月 18 日，例行檢查 Shield 狀態，發現 `threats.db` 裡的寫入時間不對。

往前查：**三週，零寫入。**

調查原因：

| 問題 | 真相 |
|------|------|
| CatBoost 評分 | ❌ 從未載入（sklearn ABI 版本不一致） |
| Shield 監控 log | ❌ 指向空檔案（3e7-access.log 是空的） |
| ML 封鎖決策 | ❌ 全靠 rule-based fallback |

**我的 ML WAF 跑了三週，ML 從來沒有跑過。**

那些漂亮的偵測率、bypass 數字——全是規則引擎在撐，不是 AI。

---

## 真正的驗收

重新訓練 CatBoost（在 container 內，F1=1.0），修好 log 路徑，讓龍蝦再打一輪：

```
17:32:53  172.16.32.28  sensitive_scan  score=1.00  banned
17:33:23  172.16.32.28  sensitive_scan  score=1.00  banned
17:33:53  172.16.32.28  sensitive_scan  score=1.00  banned
```

龍蝦的 ban_guard 也回報：

```json
"ban_guard": {
  "banned": true,
  "consecutive_fails": 2,
  "ban_trigger": { "status_code": 403 }
}
```

兩邊同時確認：Shield 封了龍蝦，龍蝦確認自己被封了。

**從 5 月 18 日起，家裡的 Shield 才是真正在運作的防護。**

---

## 我學到了什麼

**假安全感比沒安全更危險。**

如果我沒有放龍蝦來打，我永遠不會發現 CatBoost 沒在跑。log 看起來正常，偵測率看起來正常，一切都很好——只是 ML 從來沒有工作過。

傳統的測試方法不會發現這種問題。你寫 unit test，測的是邏輯；你看 log，看到的是有沒有 error。但「模型載入失敗靜默 fallback 到規則引擎」這種問題，只有在你真的去攻擊它、去驗收結果的時候才會冒出來。

**讓另一個 AI 打你自己的系統，是目前我找到最有效的驗證方式。**

---

龍蝦還有 A-L4（黑盒模型探測）和 A-L5（對抗樣本）沒用出來。

Shield 的 L2 session 序列分析還在開發。

這場對打還沒結束。

