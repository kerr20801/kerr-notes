# K8s 常見的真實錯誤 — 邊學邊做留下的痕跡

> 這篇不是理論，是實際環境裡看到的問題。每個錯誤背後都有一個「當時不知道」，記下來給自己和別人參考。

---

## 錯誤一：3 個 Master，但沒有 VIP

**症狀：** 3 台 Master（.11/.12/.13），kubeconfig 和所有 worker 的 API Server 指向固定的 .11。

**問題：** .11 一掛，API Server 就消失了。.12 和 .13 還活著，etcd quorum 還在（2/3 存活），但沒有人能連進去——因為所有人都只知道 .11 在哪裡。

**誤解：** 3 個 Master 的意義是 etcd quorum——掛一台，cluster 狀態還在，另外兩台可以接手。但「可以接手」的前提是有人連得進去。沒有 VIP，HA 是假的，只是多了幾台浪費資源的機器。

**正確做法：** Keepalived 做一個 VIP（比如 .10），kubeconfig 和所有 worker 指向 VIP。.11 掛了，VIP 自動漂到 .12，cluster 繼續運作，使用者完全無感。

```
VIP (.10) ← Keepalived
  ├── Master .11 (MASTER)
  ├── Master .12 (BACKUP)
  └── Master .13 (BACKUP)
```

這是 3 個 Master 的標配，沒做就不算真正的 HA。

---

## 錯誤二：有 Namespace，但沒有 ResourceQuota

**症狀：** Namespace 有分，各服務和各部門分開了，看起來很整齊。

**問題：** 搶資源的問題完全沒解決。Namespace 是邏輯隔離，不是資源隔離。A 部門的 Pod 瘋狂吃 CPU，B 部門的服務就跟著變慢，兩個 Namespace 但互相影響。

**正確做法：** Namespace + ResourceQuota 才是完整的隔離。

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-a-quota
  namespace: team-a
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
```

每個 Namespace 設好上限，超過就排隊等，不會無限制搶其他人的資源。

---

## 錯誤三：Ingress 路由繞錯方向

**症狀：** Ingress Controller 的流量路徑設計成：

```
外部流量 → VIP (.10) → Ingress → 又繞回 .10 → ...
```

**問題：** 邏輯上繞了一圈，流量路徑不清楚，debug 困難，而且有潛在的迴圈風險。

**正確做法：** 流量路徑應該是單向的：

```
外部流量 → LoadBalancer VIP (.9，MetalLB 管理)
              → Ingress Controller
                → Service
                  → Pod
```

Ingress 的 VIP 和 Master 的 VIP 分開，各管各的。Master VIP 給 API Server 用，Ingress VIP 給服務流量用，兩個不混。

---

## 錯誤四：Runner 定位錯了

**症狀：** K8s 上跑著 GitLab Runner，但用途是「取代原本的 GitLab Runner」，而不是「在 K8s 上動態分配資源跑 job」。

**問題：** 這樣的 Runner 跑在 K8s 上，但沒有用到 K8s 的任何優勢。動態起 Pod、用完就刪、資源按需分配——這些全部沒有。只是把一個固定的 executor 搬進去而已。

**K8s Runner 真正的用途：**

```yaml
# 每個 job 動態起一個 Pod
# job 結束 Pod 刪除
# 不同部門的 job 跑在不同 Namespace
# ResourceQuota 控制各部門能用多少資源
```

各部門的 pipeline 在自己的 Namespace 跑，資源隔離，不互搶，這才是 K8s Runner 的正確定位。

---

## 這些錯誤的共同點

**沒有問「為什麼」就開始做。**

- 3 個 Master：知道要 HA，但沒想過 HA 的前提是有人連得進去
- Namespace：知道要隔離，但沒想過隔離的是什麼
- Ingress 路由：知道要有 VIP，但沒想清楚流量路徑
- Runner：知道要上 K8s，但沒想過上去要得到什麼

K8s 的概念多、元件多，很容易「知道這個東西存在」但「不知道它解決什麼問題」。這個差距，只有在出問題之後才會真正補上。

**邊學邊做不是壞事，但要知道自己在哪個階段。** 出了問題、查了原因、修好了，這才算真正學會了一個東西。

---

*這篇記的是錯誤，下一篇談工具選型：Calico + NGINX Ingress 和 Cilium + Gateway API 兩個組合的比較。*
