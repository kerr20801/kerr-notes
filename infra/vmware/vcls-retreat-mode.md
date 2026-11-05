# vCLS VM 刪不得：用 Retreat Mode 正確停用 vSphere Clustering Services

> **環境**：vSphere 7.x，ESXi 兩節點叢集，HDD datastore

---

## 症狀

某天 ESXi host 開始出現以下狀況：

- vCenter EAM（ESX Agent Manager）持續報 `InsufficientClusterResources`
- DRS 莫名 suspend 叢集內的 VM
- HDD datastore I/O 異常高，幾乎把 host 凍結
- vCenter 畫面卡頓，SSH 還能進但 `vim-cmd` 回應很慢

重啟 EAM service 沒用。重啟 vCenter 沒用。

---

## 根本原因

vSphere 7.x 引入了 **vCLS（vSphere Clustering Services）**，會自動在叢集內建立幾個小 VM 來維持 DRS/HA 功能。

如果你直接把這些 vCLS VM 刪掉——不管是從 vCenter UI 刪，還是從 ESXi 直接 `vim-cmd vmsvc/destroy`——EAM 不會死心：

```
EAM 偵測到 vCLS VM 不見了
  → 每 30 秒嘗試重建
  → 重建需要在 datastore 寫入大量 checkpoint 檔案
  → HDD I/O 被打滿
  → host 所有 VM 受到波及
  → DRS 判斷資源不足，開始 suspend VM
  → 情況越來越糟
```

這是一個死循環，而且越等越嚴重。

---

## 錯誤的處理方式

- ❌ 繼續刪 vCLS VM → EAM 繼續重建，I/O 繼續爆
- ❌ 重啟 EAM service → 治標，服務起來還是會繼續重試
- ❌ 重啟 vCenter → 同上
- ❌ 等它自己好 → 不會好

---

## 正確解法：Retreat Mode

vSphere 提供 **Retreat Mode** 讓你正確停用 vCLS，不是刪 VM 而是告訴 EAM「這個叢集不需要 vCLS 了」。

設定後 EAM 立刻停止重試，已存在的 vCLS VM 會被自己清掉。

### Step 1：取得 Cluster 的 MoRef ID

透過 vCenter REST API：

```bash
curl -sk -u 'administrator@your.domain:YourPassword' \
  -X GET "https://vcenter-ip/rest/vcenter/cluster" \
  -H "vmware-api-session-id: $(curl -sk -u 'administrator@your.domain:YourPassword' \
    -X POST 'https://vcenter-ip/rest/com/vmware/cis/session' | python3 -c \
    'import sys,json; print(json.load(sys.stdin)["value"])')"
```

回傳會有 `cluster` 欄位，格式是 `domain-cXX`，記下這個 ID。

### Step 2：設定 Retreat Mode

透過 vCenter SOAP API 的 `UpdateClusterProfile` 或直接用 `Advanced Settings`：

**方法 A：vCenter UI（最簡單）**

1. vCenter → 選叢集 → Configure → Configuration → Advanced Settings
2. 新增 key：`config.vcls.clusters.domain-c{ID}.enabled`
3. 值設為：`false`
4. 儲存

**方法 B：Python（適合自動化）**

```python
from pyVim.connect import SmartConnect
from pyVmomi import vim
import ssl

context = ssl.SSLContext(ssl.PROTOCOL_TLSv1_2)
context.verify_mode = ssl.CERT_NONE

si = SmartConnect(
    host='vcenter-ip',
    user='administrator@your.domain',
    pwd='YourPassword',
    sslContext=context
)

content = si.RetrieveContent()
# 找叢集
for dc in content.rootFolder.childEntity:
    for cluster in dc.hostFolder.childEntity:
        if hasattr(cluster, 'configurationEx'):
            cluster_id = cluster._moId  # e.g. "domain-c8"
            
            spec = vim.cluster.ConfigSpecEx()
            spec.dasConfig = cluster.configurationEx.dasConfig
            # 設定 retreat mode key
            option = vim.option.OptionValue()
            option.key = f'config.vcls.clusters.{cluster_id}.enabled'
            option.value = 'false'
            
            cluster.ReconfigureComputeResource_Task(spec=spec, modify=True)
```

### Step 3：確認

設定後約 2-5 分鐘：
- EAM 停止 `InsufficientClusterResources` 報錯
- HDD I/O 恢復正常
- 已存在的 vCLS VM 會被 EAM 自行移除（不用你手動刪）

---

## 恢復 vCLS

把同一個 key 改回 `true` 即可，EAM 會重新建立 vCLS VM。

---

## 為什麼這個坑這麼容易踩

vCLS 是 vSphere 7.0 才加入的功能，官方文件對「怎麼停用」的說明不夠清楚。

搜尋到的文章大多是說「刪掉 vCLS VM」——這是錯的。而且刪完之後短時間內看起來好像沒事，等 EAM 開始瘋狂重建才知道出了問題。

如果你的 datastore 是 SSD，I/O 夠快，可能感覺不明顯。但 HDD 環境下，這個問題會把 host 整個拖垮。

---

## 相關指令備查

```bash
# 列出叢集內所有 VM（在 ESXi 上）
vim-cmd vmsvc/getallvms

# 查看特定 VM 的電源狀態
vim-cmd vmsvc/power.getstate {vmid}

# 查看 datastore I/O（判斷是否異常）
esxtop  # 進去後按 d 切換到 disk 視圖
```

---

---

## 連帶影響：ELK 叢集無故降級

同一個事件裡，ELK 是第一個出症狀的，反而讓排查方向走錯。

### ELK 當時的症狀

- Elasticsearch cluster health 從 green 變 yellow，部分 shard unassigned
- Kibana 查詢 timeout，Logstash pipeline 開始 backpressure
- `_nodes/stats` 看到 indexing latency 飆高
- JVM heap 正常，shard 數量也沒超限

直覺會往 ELK 本身找問題：heap 不夠？shard 太多？mapping 爆炸？都不是。

### 真正的原因

ELK 的 VM 跑在同一個 HDD datastore 上。vCLS EAM 瘋狂寫 checkpoint 把 HDD I/O 打滿，ELK VM 的磁碟寫入被擠掉，Elasticsearch 的 translog flush 跟不上，shard 開始出問題。

```
EAM checkpoint write 佔滿 HDD I/O
  → ELK VM 磁碟寫入 latency 暴增
  → Elasticsearch translog flush 超時
  → shard 標記為 unassigned
  → cluster health yellow
```

Elasticsearch 沒有壞，是底層 VM 的 storage 被別人搶完了。

### 怎麼確認是 storage 層問題

在 ESXi 上跑 `esxtop`，切到 disk 視圖（按 `d`），看 `DAVG`（device average latency）：

- 正常：< 5ms
- 有問題：> 20ms，甚至 > 100ms

如果 DAVG 異常高，問題不在 Elasticsearch，在 datastore。

### 排查順序建議

ELK 在 VM 上跑、cluster 無故降級時，先確認這幾層再進 Elasticsearch 內部找問題：

1. `esxtop` 確認 datastore latency 正常
2. vCenter 確認沒有其他 VM 或 service 在大量寫同一個 datastore
3. EAM log 確認沒有瘋狂重試（`/var/log/vmware/eam/eam.log`）
4. 都正常才進 `_cluster/health`、`_nodes/stats` 找 Elasticsearch 本身的問題

---

*記錄於 2026-05-19，vSphere 7.x 環境實際排查案例*
