# K8S kubelet 憑證自動輪替，這次沒生效

> **環境**：kubeadm 建的 3-master HA 叢集，跑 Mattermost / Outline / HackMD / GitLab Runner

---

## 症狀

Mattermost 登入頁點「GitLab SSO」按鈕，跳出「從 Discovery Document 取得端點時發生錯誤」。同時 HackMD 也打不開。

登入頁本身正常顯示（不用查外部服務），只有牽涉到外部整合的功能壞掉——這個現象本身就是排查方向的第一個線索。

---

## 排查鏈

**Step 1：SSO 按鈕實際在打什麼**

點下去會打 `https://gitlab.xxx/.well-known/openid-configuration`。查 Mattermost pod log：

```
DNS lookup failed: connection refused / i/o timeout
```

打的是 cluster CoreDNS（`10.245.0.10:53`），不是外部 DNS 掛了。

**Step 2：CoreDNS 狀態**

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
# 4 個 replica，只有 2 個 Running，另外 2 個 Pending
```

`kubectl describe` 那兩個 Pending 的：

```
FailedScheduling: node affinity 限定在 master 節點，其中一台 unreachable
```

**Step 3：Node 狀態**

```bash
kubectl get nodes
# master-1   Ready     control-plane
# master-2   Ready     control-plane
# master-3   Unknown   control-plane   ← 這個
```

`master-3` 從前一天下午開始 `Ready=Unknown`（kubelet 停止回報），已經卡了超過一天沒人發現——因為另外兩台 master 撐住了叢集運作，只有依賴 CoreDNS 滿容量的功能（外部 DNS 查詢）才會出事，一般的內部服務完全正常，所以完全沒有症狀提示node掛了。

---

## 根因

上 `master-3` 看 kubelet：

```bash
journalctl -u kubelet -n 50
```

```
E bootstrap.go:266] part of the existing bootstrap client certificate
  in /etc/kubernetes/kubelet.conf is expired: 2026-07-31 16:00:11 +0000 UTC
E run.go:74] "command failed" err="failed to run Kubelet: unable to load
  bootstrap kubeconfig: stat /etc/kubernetes/bootstrap-kubelet.conf:
  no such file or directory"
```

kubelet 跟 API server 做 mTLS 認證用的 **client 憑證**過期了。這個憑證正常應該在到期前被 kubelet 內建的 cert-rotation 機制自動換新——這次沒換成功，systemd 陷入無限重啟（restart counter 7000+ 次），因為：

1. 憑證過期
2. 想重新走 bootstrap 流程去要新憑證
3. 但 `bootstrap-kubelet.conf` 這個檔案在 node **正常 join 完成後就會被清掉**（設計如此，join 完不需要留著）
4. 沒有這個檔案，kubelet 沒有 fallback 手段能重新拿憑證，卡死

**這次沒查出來的事**：為什麼內建的自動輪替沒能在到期前生效。可能是那段時間節點網路有異常、也可能是 CSR 簽核流程卡住，沒有繼續往下挖——這是止血優先的修法，根本原因留了一個問號。

> **後續更新**：這個問號後來被查出來了，而且答案比「網路異常」嚴重得多——是一年前一次「先讓它動起來」的緊急修復，順手關掉了自動輪替、還放大了權限。完整故事見 [`05-quick-fix-becomes-next-years-root-cause.md`](05-quick-fix-becomes-next-years-root-cause.md)。

---

## 修法

在健康的 master 上產生新的 join token：

```bash
sudo kubeadm token create --print-join-command
# kubeadm join <VIP>:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>
```

在故障節點上：

```bash
sudo systemctl stop kubelet
sudo mv /etc/kubernetes/kubelet.conf /etc/kubernetes/kubelet.conf.bak
sudo rm -f /var/lib/kubelet/pki/kubelet-client*
sudo kubeadm join phase kubelet-start <VIP>:6443 \
  --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>
sudo systemctl restart kubelet
```

關鍵是 `kubeadm join phase kubelet-start`——只執行完整 join 流程裡「kubelet TLS bootstrap」這一小段，**不會碰 etcd member、不會動 control-plane 的 static pod manifest**。CA 憑證的 hash 前後兩次產生 token 都一樣（只有 token 換新，CA 本身沒變），可以放心跑。

跑完 node 轉回 `Ready`，原本卡 Pending 的兩個 CoreDNS pod 自動排程上去，SSO 跟 HackMD 都恢復。

---

## 快速排查順序（這條鏈值得記下來）

外部整合/SSO/webhook 突然失敗，但網站本身正常：

```bash
# 1. DNS 解析是不是有問題（cluster CoreDNS，不是外部 DNS）
kubectl get pods -n kube-system -l k8s-app=kube-dns

# 2. CoreDNS 容量不足，通常是某個 node NotReady/Unknown 拖累排程
kubectl get nodes

# 3. 該 node 的 kubelet 狀態
journalctl -u kubelet -n 50
```

「網站正常、只有外部整合壞掉」→「DNS 解析失敗」→「CoreDNS 容量不足」→「某個 master NotReady」這條鏈，這次證明是有效的排查路徑，比直接去猜 SSO 設定本身壞了快很多。

---

## 順手做的事：補一個主動偵測

這次是靠使用者回報症狀才發現 node 已經掛了一天多。事後補了一支 watchdog，每 5 分鐘查一次 node Ready 狀態跟 CoreDNS pod 健康，異常就通知——**只偵測+通知，不自動修復**（沒有直接操作 K8s control-plane 節點的權限/工具鏈，這類修復目前還是要人工上機處理）。

下次同樣的事再發生，應該 5 分鐘內就會發現，不用等別人先撞到症狀。
