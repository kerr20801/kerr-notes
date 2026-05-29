# Nginx 三層自動 Failover：K8s → K3S → Docker DR

> 不是最標準的做法，但它能跑，而且故障切換是自動的。

## 架構概覽

```
Client
  ↓
NGINX (192.168.1.43/.44, Keepalived VIP)
  ↓ proxy_next_upstream
K8s / Traefik (192.168.1.78:443)   ← 正常流量
  ↓ 502/503/504
K3S (192.168.1.51:443)             ← K8s 掛掉自動切
  ↓ 502/503/504
Docker DR (192.168.1.28:443)       ← K3S 也掛才到這
```

三個後端用 `proxy_next_upstream` 串起來，nginx 偵測到 upstream 回 5xx 或 timeout 就自動往下一層走，不需要 health check daemon、不需要人工介入。

---

## Nginx 核心設定

```nginx
# upstream 定義三層後端
upstream k8s_cluster {
    server 192.168.1.78:443 max_fails=1 fail_timeout=30s;
}

upstream uat_k8s {
    server 192.168.1.51:443 max_fails=1 fail_timeout=5s;
}

upstream docker_dr {
    server 192.168.1.28:443 max_fails=1 fail_timeout=5s;
}

server {
    listen 443 ssl;
    # ... ssl_certificate etc.

    # nginx-shield 認證層（auth_request）
    location /_shield_check {
        internal;
        proxy_pass http://127.0.0.1:9998/check;
    }

    location / {
        auth_request /_shield_check;

        proxy_pass https://k8s_cluster;
        proxy_ssl_verify off;
        proxy_connect_timeout 500ms;    # 快速失敗，不等 K8s 慢慢 timeout

        # 遇到這些錯誤才往下切，不包含 5xx application error
        proxy_next_upstream error timeout http_502 http_503 http_504;

        error_page 502 503 504 = @uat_k8s;
    }

    # 第二層：K3S
    location @uat_k8s {
        proxy_pass https://uat_k8s;
        proxy_ssl_verify off;
        proxy_connect_timeout 500ms;
        proxy_next_upstream error timeout http_502 http_503 http_504;

        error_page 502 503 504 = @dr_mode;
    }

    # 第三層：Docker DR
    location @dr_mode {
        proxy_pass https://docker_dr;
        proxy_ssl_verify off;
    }
}
```

---

## 關鍵設計決策

### `proxy_connect_timeout 500ms`
不設這個，K8s 如果 node 掛掉但 TCP 還沒斷，nginx 會等到 `proxy_read_timeout`（預設 60s）才認輸。設 500ms 讓切換在 1 秒內完成，用戶幾乎感覺不到。

### `max_fails=1 fail_timeout=30s`
K8s 第一次失敗就標記不可用，30 秒內直接送 K3S，不再試 K8s。避免每個請求都要等 500ms timeout 才切。

### `proxy_next_upstream error timeout http_502 http_503 http_504`
只切換 gateway 層的錯誤，不切換 `http_500`（應用程式錯誤）。如果 K8s 回 500，那是 app 的問題，K3S 跑一樣的 image 也會回 500，切換沒意義。

### `@location` 串接 vs `error_page` 遞迴
nginx `@named_location` 不支援再次觸發 `error_page` 遞迴，所以每層要明確寫 `error_page` 指向下一個 `@location`。三層就是三段，沒有更優雅的寫法。

---

## K3S 吃 K8s 資料的問題

K3S 是作為 K8s 的 failover，理想上應該吃同一份 DB/Storage。實作上：

- **Stateless 服務**：沒問題，K3S 跑一樣的 image，直接起
- **有狀態服務**：K3S PVC 指向同一個 NAS（NFS mount），K8s 寫入的資料 K3S 能讀到

限制是 K8s 和 K3S 不能同時寫同一個 PVC（RWO volume），所以 failover 後 K8s 如果還沒完全掛，可能有競爭寫入的問題。這個 lab 環境接受這個風險，production 環境要考慮用 RWX storage 或 distributed lock。

---

## nginx-shield 整合

```nginx
location /_shield_check {
    internal;
    proxy_pass http://127.0.0.1:9998/check;
    proxy_pass_request_body off;
    proxy_set_header Content-Length "";
    proxy_set_header X-Original-URI $request_uri;
}
```

`auth_request` 在進入任何一層 upstream 前都會先打 `/_shield_check`。如果 shield 回 401/403，nginx 直接拒絕，不會打到後端。Shield 掛掉（500）會讓所有請求都失敗，所以 shield 本身要另外保持高可用。

---

## 這個方案適合的場景

- Lab / staging 環境想要自動 failover 但不想維護 L4 load balancer
- K8s 和 K3S 同時在線、版本差不多、跑一樣的服務
- 可以接受 failover 時 in-flight 請求失敗（非 zero-downtime）

不適合：
- Production 需要 zero-downtime 的場合 → 用 L4 LB + proper health check
- K8s 和 K3S 版本差異大，API 不相容
- 需要 session affinity（failover 後 session 一定斷）
