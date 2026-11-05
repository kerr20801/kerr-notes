# Chrome MV3 Service Worker 三個坑

> **背景**：把 Chrome 套件從 MV2 改 MV3，或直接新寫 MV3 套件，一定會遇到的問題。

---

## 坑一：sendMessage 不能用 Promise，要用 callback

MV2 時代這樣寫沒問題：

```javascript
// ❌ MV3 Service Worker 裡會壞掉
const response = await chrome.tabs.sendMessage(tabId, { type: 'ping' });
```

MV3 的 Service Worker 用 Promise 版的 `sendMessage`，在某些情況下 response 永遠不會回來，訊息直接掉了。

**正確寫法**：callback 風格。

```javascript
// ✅ 用 callback
chrome.tabs.sendMessage(tabId, { type: 'ping' }, (response) => {
  void chrome.runtime.lastError; // 吃掉 "receiving end does not exist" 錯誤
  if (response) doSomething(response);
});
```

Storage API 同樣的問題：

```javascript
// ❌
const data = await chrome.storage.local.get('key');

// ✅
chrome.storage.local.get('key', (data) => {
  // 在這裡處理
});
```

---

## 坑二：onMessage listener 要 return true

Content script 收到訊息後，如果需要非同步回應（比如要等 storage 讀完才能 reply），listener 必須 `return true`，否則 message channel 會被關掉，`sendResponse` 永遠送不出去。

```javascript
// ❌ 非同步回應送不出去
chrome.runtime.onMessage.addListener((msg, sender, sendResponse) => {
  chrome.storage.local.get('setting', (data) => {
    sendResponse({ ok: true, data });
  });
  // 沒有 return true，channel 已關閉
});

// ✅
chrome.runtime.onMessage.addListener((msg, sender, sendResponse) => {
  chrome.storage.local.get('setting', (data) => {
    sendResponse({ ok: true, data });
  });
  return true; // 保持 channel 開著
});
```

如果不需要非同步回應，就 `return false` 或不 return。

---

## 坑三：Content Script 啟動時 Service Worker 可能還沒醒

MV3 的 Service Worker 是 event-driven，閒置一段時間會被瀏覽器睡掉。

Content script 一 inject 就馬上發 `sendMessage` 給 background，很容易踢到 SW 還在喚醒中的空窗期，訊息掉了。

**解法**：加 retry。

```javascript
// content.js
function initWithRetry(retries = 3) {
  chrome.runtime.sendMessage({ type: 'get_settings' }, (response) => {
    if (chrome.runtime.lastError || !response) {
      if (retries > 0) {
        setTimeout(() => initWithRetry(retries - 1), 150);
      }
      return;
    }
    applySettings(response);
  });
}

initWithRetry();
```

150ms 的 delay 通常夠 SW 醒過來。

---

## 快速 checklist

寫 MV3 background (Service Worker) 時：
- [ ] `chrome.storage` 用 callback，不用 await
- [ ] `chrome.tabs.sendMessage` 用 callback，不用 await
- [ ] 所有 `onMessage` listener 非同步的都有 `return true`
- [ ] `void chrome.runtime.lastError` 吃掉預期內的錯誤

寫 content script 時：
- [ ] 啟動時發給 SW 的訊息有 retry 機制（150ms，3次）
- [ ] 同步的 listener 有 `return false` 或直接不 return

---

*記錄於 2026-05，開發 SentinelDLP / EnvGuard / PIW 套件時實際遇到的問題*
