# EnvGuard：Chrome 套件解決「錯環境誤操作」問題

## 問題

8 個 tab 全部長得差不多——stock-ai dashboard、Outline、staging 測試環境、公司內網。

一不小心在 production 按了不該按的按鈕。

這個問題沒有技術門檻，是純粹的人眼辨識問題。解法也不需要複雜——**在視覺上讓環境之間有明顯差異**就夠了。

---

## 設計決策

### Shadow DOM 色條，而非注入 CSS

直接在 `<body>` 插 `<div style="...">` 會被頁面的 CSS 蓋掉——`overflow:hidden`、`reset.css`、框架的全局樣式都可能讓色條消失或變形。

解法：用 `attachShadow({ mode: 'closed' })` 建立隔離的 DOM 樹，頁面的所有 CSS 規則都不會穿透進來。

```javascript
const host = document.createElement('div');
const shadow = host.attachShadow({ mode: 'closed' });
// 在 shadow 裡面的樣式完全隔離
```

所有 style 直接寫在 `style.cssText`，不依賴外部 stylesheet，避免 CSP 衝突。

### Unknown 狀態顯示灰條，不是直接隱藏

初版設計：認不出環境就不顯示任何東西。

問題：用戶不知道是「這頁是 PROD 沒顏色」還是「套件沒偵測到」。沉默比錯誤更難排查。

改法：Unknown 顯示灰色細條，讓用戶知道「EnvGuard 看到了但認不出來」。可以在 popup 關掉這個行為。

### 三層偵測，順序有意義

```
Layer 1  用戶自訂規則     最高優先，明確覆蓋
Layer 2  heuristics        localhost / dev ports / staging patterns
Layer 3  path keywords     /staging/ /uat/ /preprod/
```

Layer 1 用精確 substring match，不是 regex——減少誤觸發，也讓用戶容易理解規則效果。

### SPA 支援

傳統的 `DOMContentLoaded` 在 SPA 裡只觸發一次，之後的路由切換不會重跑。

```javascript
const _push = history.pushState.bind(history);
history.pushState = (...a) => { _push(...a); setTimeout(run, 150); };
window.addEventListener('popstate', () => setTimeout(run, 150));
```

150ms delay 讓 SPA framework 有時間更新 DOM 再重新偵測。

---

## 偵測邏輯

| 環境 | 偵測條件 |
|------|---------|
| DEV | localhost / 127.x / 192.168.x / *.local / dev. / port 3000/8080/5173 等 |
| STAGING | staging. / uat. / preprod. / qa. / sandbox. / test. in hostname；/staging/ in path |
| PROD | prod. / production. / live. in hostname；或用戶手動標記 |
| Unknown | 以上都不符合 |

色條：PROD 紅 / STAGING 橘 / DEV 藍 / Unknown 灰

---

## Popup 功能

- 顯示偵測結果 + 哪一層偵測到（auto-detected / custom rule）
- 手動標記目前 domain
- 白名單（不在此 domain 顯示）
- Unknown 灰條開關

---

## Roadmap

**v0.2**：tab title prefix（`[PROD] Dashboard`）、wildcard 規則、內建雲端 console 規則（AWS/GCP/Azure）

**v0.3**：ML opt-in 層——~8MB on-device classifier，偵測 URL heuristics 認不到的情況（random deploy preview URL、頁面內容有 PROD WARNING banner）。用戶明確選擇才下載，大小透明，不走 Google 那條「靜默下載 4GB」的路。

---

## Repo

`github.com/kerr20801/EnvGuard` — Chrome MV3，純前端，無後端
