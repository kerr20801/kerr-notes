# Dex + Outline on K8S：session 斷線 / no access / 404 三個獨立根因

> **環境**：Outline wiki 跑在 K8S，OIDC 認證透過 Dex

症狀看起來像同一個問題（一下 no access、一下 404、重新整理就好），
其實是三個完全不同的根因同時發生。

---

## 根因一：Collection permission = NULL

Outline 建 collection 沒有明確設定 permission 時，預設是 `NULL`。
`NULL` 不是「所有人可讀」，是「需要明確加入成員才能存取」。

結果：OIDC 登入的帳號在 `read_write` 和 `NULL` 的 collection 之間切換文件，
間歇性出現 `no access to this doc`，但重新整理就好（因為跳回有權限的 collection）。

**確認**：
```bash
kubectl exec -n outline postgres-0 -- psql -U outline -c \
  "SELECT name, permission FROM collections ORDER BY permission NULLS FIRST;"
```

**修法**：
```sql
UPDATE collections SET permission = 'read_write' WHERE permission IS NULL;
```

---

## 根因二：Dex authRequests 預設只有 10 分鐘

Outline 登入流程：
1. 使用者點登入 → Dex 產生 `state` 存在 **memory**
2. 使用者輸入帳密 → callback 帶 state 回來驗證

Dex 預設 `authRequests` 有效期 **10 分鐘**。超時或 Dex pod 重啟（memory 清空）→
`No state was available after OAuth flow` → redirect 到 404。

**修法**：調整 Dex ConfigMap：

```yaml
expiry:
  signingKeys: "24h"    # 預設 6h，太短會讓舊 session 失效
  idTokens: "48h"       # 預設 24h
  authRequests: "30m"   # 預設 10m，這個最關鍵
```

**注意**：Dex 用 `storage: memory` 時，pod 重啟後所有 session state 清空，
所有人需要重新登入。這是設計行為，不是 bug。
要避免這個就換 `storage: kubernetes`，把 state 存 CRD。

---

## 根因三：沒有 refresh token（offline_access scope 沒加）

這個最難找，因為症狀是「看文件中途突然 no access」，跟第一個症狀一模一樣。

原因：Outlook 的 access token 有效期有限。Token 過期時，Outline 需要靜默換新 token，
這需要 **refresh token**。但 refresh token 只有在請求 `offline_access` scope 時才會發出。

`OIDC_SCOPES` 預設只有 `openid profile email`，沒有 `offline_access`。
→ 沒有 refresh token → token 過期 → API 403 → 前端顯示 `no access`。

**確認目前的 scope**：
```bash
kubectl get deployment -n outline outline -o jsonpath='{.spec.template.spec.containers[0].env}' \
  | python3 -c "
import sys, json
for e in json.load(sys.stdin):
    if 'OIDC' in e['name']:
        print(e['name'], '=', e.get('value',''))
"
```

**修法**：在 Outline deployment 加上 `offline_access`：
```
OIDC_SCOPES=openid profile email offline_access
```

修改後需要重新登入一次，新 scope 才會生效。

---

## 快速排查順序

遇到 no access / 404 / session 斷線，依序確認：

```bash
# 1. Collection permission 有無 NULL
kubectl exec -n outline postgres-0 -- psql -U outline -c \
  "SELECT name, permission FROM collections ORDER BY permission NULLS FIRST;"

# 2. Dex expiry 設定
kubectl get configmap -n outline dex-config -o yaml | grep -A5 expiry

# 3. Outline OIDC_SCOPES 有無 offline_access
kubectl get deployment -n outline outline -o yaml | grep -A1 OIDC_SCOPES
```

資源正常（CPU < 5%、RAM < 35%）時出現這些症狀，幾乎都是認證/權限層的問題，
不是資源問題，不要往那個方向查。
