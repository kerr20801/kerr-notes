# K8S Node 重啟後 NotReady

> **環境**：Ubuntu 24.04，containerd，kubeadm 建叢集，跑在 vSphere VM 上

---

## 症狀

Node 重啟後 `kubectl get nodes` 顯示 NotReady，等很久還是不恢復。

```bash
kubectl get nodes
# NAME       STATUS     ROLES    AGE
# k8s-master Ready      master   30d
# k8s-node1  NotReady   <none>   30d   ← 重啟後卡在這
```

SSH 進去 node 本身，服務看起來都在跑，但 kubelet 狀態不對。

---

## 排查順序

### Step 1：看 kubelet 狀態

```bash
systemctl status kubelet
journalctl -u kubelet -n 50 --no-pager
```

最常看到的錯誤訊息：

**A. br_netfilter 沒載入**
```
failed to run Kubelet: running with swap on is not supported...
# 或
sysctl: cannot stat /proc/sys/net/bridge/bridge-nf-call-iptables: No such file or directory
```

**B. containerd socket 還沒準備好**
```
unable to determine runtime API version: rpc error: code = Unavailable
desc = connection error: ... /run/containerd/containerd.sock
```

**C. 兩個都有**：先處理 A，再看 B。

---

### Step 2A：br_netfilter 沒有在開機時載入

K8S 要求 `br_netfilter` 模組，讓 iptables 能看到 bridge 的流量。這個模組重啟後不會自動載入，除非明確設定。

**確認**：
```bash
lsmod | grep br_netfilter
# 沒有輸出 = 沒載入
```

**永久修復**：

```bash
# 開機自動載入模組
cat <<EOF > /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

# sysctl 設定永久生效
cat <<EOF > /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# 立刻套用（不用重啟）
modprobe overlay
modprobe br_netfilter
sysctl --system
```

之後重啟 kubelet：
```bash
systemctl restart kubelet
```

---

### Step 2B：containerd 還沒起來 kubelet 就先跑了

systemd 的啟動順序問題。kubelet 依賴 containerd，但有時 containerd socket 還沒準備好，kubelet 就先嘗試連線，失敗後不會自動重試。

**確認**：
```bash
systemctl status containerd
# 如果是 active，但 kubelet 還是不行
ls /run/containerd/containerd.sock
```

**手動修復（當下）**：
```bash
systemctl restart kubelet
```

通常 restart 一次就好，因為這時 containerd 已經起來了。

**根本修復**：確認 kubelet 的 systemd unit 有等 containerd：

```bash
cat /lib/systemd/system/kubelet.service
# 應該要有 After=containerd.service 或 After=cri-docker.service
```

如果沒有，加上去：
```bash
mkdir -p /etc/systemd/system/kubelet.service.d/
cat <<EOF > /etc/systemd/system/kubelet.service.d/10-containerd.conf
[Unit]
After=containerd.service
Requires=containerd.service
EOF

systemctl daemon-reload
```

---

## vSphere VM 特有的問題

在 vSphere 上，VM 重啟有時候比實體機慢，VMware Tools 初始化會多花幾秒。這段時間 network interface 可能還沒完全起來，kubelet 掃描網路設定時找不到預期的 interface。

症狀：kubelet log 裡看到 `failed to get node IP`。

```bash
journalctl -u kubelet | grep "node IP"
```

如果有這個，通常等 30 秒再 `systemctl restart kubelet` 就好。要永久解決：

```bash
# 在 kubelet 設定裡指定 node IP，不讓它自己猜
cat <<EOF >> /etc/default/kubelet
KUBELET_EXTRA_ARGS=--node-ip=你的VM實際IP
EOF
systemctl daemon-reload && systemctl restart kubelet
```

---

## 快速確認叢集恢復

```bash
# Node 回到 Ready
kubectl get nodes

# Pod 都正常
kubectl get pods -A | grep -v Running | grep -v Completed
```

如果 Node Ready 但有些 Pod 還在 Pending，通常是 DaemonSet 在重新調度，等一兩分鐘自己會好。
