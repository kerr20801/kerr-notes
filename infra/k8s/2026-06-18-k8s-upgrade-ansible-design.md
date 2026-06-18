# K8s 升級自動化的架構決策

> **結論先說**：K8s 升級腳本最常見的設計失誤是做成一個大 playbook 跑到底，失敗就全部重來。正確做法是把 upgrade 拆成四個獨立可重入的 stage，每個 stage 冪等，失敗在哪就從哪繼續。再加上 `upgrade_matrix` 明確定義版本跳轉規則，整個流程就不靠「大家都知道不能跳版本」這種口耳相傳。

---

## 為什麼不能一個 playbook 跑到底

K8s 升級有個現實問題：**不能對整個叢集做 atomic rollback**。一旦某個 master node 跑完 `kubeadm upgrade apply`，那個節點就是新版本了，你不能把它「un-upgrade」。

這個現實決定了整個設計方向：

- **不能失敗就重跑整個 playbook**，因為已升級的節點重跑會出問題
- **必須能從失敗的地方繼續**，不需要手動清理狀態
- **每個 stage 必須冪等**，重跑不會損壞已升級的節點

所以架構是四個獨立 playbook：

```
01-pre-upgrade-check.yml    # 全面檢查，不改任何東西
02-upgrade-masters.yml      # 只動 master nodes
03-upgrade-workers.yml      # 只動 worker nodes
04-post-upgrade-verify.yml  # 驗收，不改任何東西
```

失敗在 02，修完繼續跑 02 就好，不需要從 01 重來。

---

## upgrade_matrix：版本跳轉圖的顯式化

K8s 有個硬規則：**不能跨 minor version 升級**。`1.30 → 1.32` 直接跑會失敗，必須走 `1.30 → 1.31 → 1.32`。

很多團隊靠口耳相傳記這個規則，過一段時間新人不知道，就炸了。

解法是把版本跳轉關係寫進 `group_vars/all.yml`：

```yaml
upgrade_matrix:
  "1.31.10":
    direct: ["1.32.6"]               # 可以直接升
    staged: ["1.32.6", "1.33.2"]     # 要到 1.33 需要先過 1.32

  "1.32.6":
    direct: ["1.33.2"]               # 直接升
    staged: ["1.33.2"]

  "1.33.2":
    direct: []                       # 已是最新，暫無升級目標
    staged: []
```

升級腳本跑之前先查這個 matrix，版本路徑不合法就直接拒絕：

```bash
./scripts/run-upgrade.sh 1.33.2  # ← 從 1.31 升到 1.33 會被擋，告訴你要先升 1.32
```

新版本出來，只改這個 YAML，不動任何 playbook 邏輯。

---

## Component 升級 vs Role 配置：兩件不同的事

把這兩件事分開是這個架構裡最關鍵的設計決策。

**Component 升級**：裝套件，升級 kubeadm/kubelet/kubectl，跑 `kubeadm upgrade`。這是技術層面的操作，可以獨立在每個節點上做，與叢集角色無關。

**Role 配置**：把某個節點重新 join 叢集（worker）、或讓 master 重新接管（control plane）。這是業務邏輯層面的操作，需要知道叢集拓撲。

舊做法常把這兩件事混在一起，結果是：組件升完但 join 失敗，你不確定要重跑整個 playbook 還是只重跑 join 部分。

新做法：

```bash
# 先升組件（冪等，可重跑）
ansible-playbook -i inventory/all k8s-component-manager.yml \
  -e target_nodes=uat_masters -e target_version=1.32.6

# 然後配置角色（獨立，失敗只重跑這段）
ansible-playbook -i inventory/all k8s-master-configurator.yml \
  -e target_masters=uat_masters -e target_version=1.32.6
```

---

## UAT 先行：K8s 升級的唯一安全網

K8s 升級在 PROD 失敗，回頭路很窄。UAT 是真正的「升級演習」，不是「測試環境也順便升一下」。

這套流程的設計讓 UAT 先行不是口號，而是強制路徑：

```
UAT masters 升級
    ↓
UAT workers 升級
    ↓ 觀察期（自行決定長度，通常1週）
    ↓
PROD masters 升級
    ↓
PROD workers 升級
```

scripts 支援 `--environment uat/prod` 參數，不加這個參數預設只跑 UAT。要跑 PROD 必須明確指定：

```bash
./scripts/run-upgrade.sh 1.32.6 --environment uat   # UAT
# ... 觀察 ...
./scripts/run-upgrade.sh 1.32.6 --environment prod  # PROD，明確行動
```

---

## Emergency Rollback 的現實

`emergency-rollback.sh` 存在，但有個必須誠實說的限制：**K8s 套件降版本在 APT 裡無法完全自動化**。

原因是 kubeadm 升級會修改 etcd schema 和 cluster config，套件降回去，config 還是新格式的。所以 rollback 腳本做的事是：

1. 自動：找最近的備份路徑
2. 自動：從備份還原 kubeadm config
3. **手動**：提示你跑套件降版本的指令

```bash
# rollback.sh 跑到這裡會印出這段，讓你自己跑
sudo apt-get install -y kubeadm=1.31.10-* kubelet=1.31.10-* kubectl=1.31.10-*
sudo systemctl daemon-reload && sudo systemctl restart kubelet
```

這不是設計缺陷，是現實限制。與其假裝可以全自動，不如明確把手動步驟列出來，讓操作者知道自己在做什麼。

---

## 兩個 Repo 的問題

這個專案原本有兩個版本：`ai-ansible-k8s-integration`（客戶展示包）和 `ansible-k8s-integration`（生產版本）。

展示包的特色：架構圖漂亮、README 詳細、roles 結構完整，但缺 `upgrade_matrix`、缺 rollback script、inventory 是硬編碼示範 IP。

生產版本的特色：`upgrade_matrix` 完整、inventory 結構有 UAT/PROD 分離、有 emergency rollback，但 README 簡陋、roles 不如展示包細。

合併方向：以生產版本為主體，把展示包的 `upgrade_matrix` 邏輯、QUICK_START、rollback script 補進來，再做 IP sanitization 後推 GitHub。

選生產版本而不是展示包的原因：**展示包設計目標是「看起來專業」，生產版本設計目標是「用起來不會炸」，這兩個不一樣**。
