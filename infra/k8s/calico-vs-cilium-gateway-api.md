# Calico vs Cilium 選型 + Kubernetes Gateway API 取代 Ingress

> 結論先說：大多數 lab 和中小型公司環境用 Calico 就夠了。Cilium 不是裝了就沒事，它的代價是真實的。

---

## Ingress 已經凍結，方向換 Gateway API

Kubernetes Ingress API 不會再加新功能了。官方已把資源投入 **Gateway API**（`gateway.networking.k8s.io`），兩者的差距：

| | Ingress | Gateway API |
|---|---|---|
| 狀態 | 凍結，只維護不擴展 | 積極開發中，v1.2+ GA |
| 角色分離 | 無，全混在一個資源 | GatewayClass（基建）/ Gateway（入口）/ HTTPRoute（路由） |
| 協定支援 | HTTP/HTTPS | HTTP、HTTPS、gRPC、TCP、TLS |
| 跨 namespace 路由 | 不支援 | 支援 |
| 權重路由 | 看 controller 支援程度 | 原生支援 |

Gateway API 的角色分離讓 infra 管 GatewayClass + Gateway，開發者只需要管 HTTPRoute，權責比較清楚。

**遷移不急，但新架構不要再用 Ingress 設計了。**

---

## CNI 選型：Calico vs Cilium

### Calico — 多數情況的正確答案

- iptables / ipvs 實作，K8s 運維人員基本都懂怎麼 debug
- 資源消耗低，node agent 輕量
- 出問題直接 `iptables -L` 、`ipset list` 就能查
- 支援 NetworkPolicy（標準），BGP peering（進階）
- 搭 Gateway API 需要另外裝 controller（Envoy Gateway、nginx Gateway Fabric 等）

### Cilium — 什麼時候才值得

- eBPF 實作，bypass iptables，高流量下延遲更低
- 內建 L7 NetworkPolicy（HTTP path、gRPC method 層級管控）
- Hubble：即時流量可觀測性，能看到 pod 間每條連線
- 自己就是 Gateway API controller，不需要另外裝
- 可以取代 service mesh（mTLS、流量加密）

### Cilium 的代價（很少文章講清楚）

**1. kernel 版本要求高**

eBPF 完整功能需要 kernel 5.10+，部分功能要 5.15+。老機器或沒在更新的 node 就直接卡住。

**2. 每個 node 的 agent 記憶體吃更多**

Cilium agent 本身比 Calico 重，加上要跑 eBPF map，在小 node（4GB RAM 以下）會很明顯。

**3. 出問題很難查**

Calico 壞掉 → `iptables -L | grep KUBE` 幾行就找到。Cilium 壞掉 → 要用 `cilium monitor`、`hubble observe`、`bpftool`，不懂 eBPF 的人會卡很久。

**4. Hubble 好用，但又是另一套要維護**

Hubble UI + Relay 是獨立元件，有自己的升級週期、資源消耗、troubleshooting。不是裝了就永遠沒事。

**5. 升級風險比 Calico 高**

eBPF program 版本和 kernel 版本有相依性，大版本升級前要確認相容矩陣，踩過的人都知道這件事不輕鬆。

---

## 實際選型判斷

```
你的環境有以下任一條件嗎？
  ├─ 單 cluster pod 數超過 5,000？
  ├─ 需要 HTTP path / gRPC method 層級的 NetworkPolicy？
  ├─ 要做 multi-cluster mesh（Cluster Mesh）？
  └─ 需要取代 Istio 做 mTLS？

全都沒有 → Calico
有其中一條 → 評估 Cilium，先確認 kernel 版本和 node 規格
```

**Lab 和中小型公司環境通常全都沒有。Cilium 的功能聽起來很吸引人，但引入的維運複雜度不划算。**

---

## Gateway API 搭配 CNI 的組合建議

| CNI | Gateway API Controller | 備註 |
|-----|----------------------|------|
| Calico | Envoy Gateway 或 nginx Gateway Fabric | 兩者都穩定，Envoy 功能更完整 |
| Calico | Traefik（支援 Gateway API） | 已有 Traefik 的環境最省事 |
| Cilium | Cilium Gateway API（內建） | 不需要另外裝，一條龍 |

Calico 環境最省事的選法：**已有 Traefik 就繼續用 Traefik，它從 v3.1 開始支援 Gateway API**，不需要換 controller。

---

## 踩坑提醒

- CNI 換掉需要重建 cluster 或至少重建所有 node，不是 `helm upgrade` 那麼簡單
- Cilium 取代 kube-proxy 是可選的（`kubeProxyReplacement: true`），但一旦開了，回頭很麻煩
- Gateway API 的 CRD 和 controller 版本要對齊，混用不同版本的 CRD 會有奇怪行為
- 不管選哪個 CNI，NetworkPolicy 的 default-deny 要從一開始就設計好，後期補很痛

---

## 實戰後記

實際跑過 Cilium 一段時間，最終退回 Calico + Ingress（現在改 Gateway API）。原因很單純：環境沒到那個規模，Cilium 的功能幾乎沒用到，但維運複雜度是真實存在的。

**不是 Cilium 不好，是環境決定工具。** 當流量規模、L7 policy 需求、或可觀測性要求真的到位，Cilium 就是對的選擇，這時候硬省才是錯的。技術選型的標準只有一個：需求，不是功能清單。
