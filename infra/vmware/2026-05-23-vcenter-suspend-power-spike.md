# vCenter Suspend 會造成電力衝擊，斷電時不能並行操作

## 背景

UPS 快沒電時，直覺是「趕快同時 suspend 所有 VM，越快越好」。  
這個直覺是錯的。

## Suspend 的實際行為

Suspend VM = 把整個 RAM 內容寫入磁碟（`.vmem` 檔案）。

- 8GB RAM 的 VM → 要寫 8GB 到 datastore
- 寫入期間：磁碟 I/O 滿載 + CPU 高使用率
- 結果：這台 VM 的**電力消耗在 suspend 期間短暫上升 10~20%**

如果同時 suspend 5 台大 VM，電力衝擊疊加。UPS 在電池已經不滿的狀態下多吃這段電，可能讓剩餘時間估算完全失準，甚至提前觸發 EMERGENCY。

## 正確做法：Sequential

一次只 suspend 一台，等完成再下一台。大 VM（>16GB RAM）之間額外等 30 秒讓 I/O 壓力消散：

```python
for name in vm_names:
    task = vm.SuspendVM_Task()
    _wait_task(task)   # 等這台完成
    # 下一台
```

```python
# 大 VM 之間加緩衝
large_vms = sum(1 for v in powered_on if v.get("mem_usage_mb", 0) > 16384)
total_sec += large_vms * 30
```

## 時間估算要夠準

Sequential suspend 的總時間取決於：

```
每台 suspend 時間 ≈ RAM(GB) / 磁碟寫入速度(GB/s) + 60s 緩衝

HDD:  0.15 GB/s → 8GB VM ≈ 53s + 60s = 113s
SATA SSD: 0.4 GB/s → 8GB VM ≈ 20s + 60s = 80s
NVMe: 1.0 GB/s → 8GB VM ≈ 8s + 60s = 68s
```

19 台 VM 的環境實測：sequential suspend 約 21 分鐘。  
所以 CRITICAL 閾值要設在 26 分鐘以上（21 + 5 buffer），不是 15 分鐘。

系統用 `shutdown_estimator` 根據當前開機的 VM 清單動態計算這個閾值，不寫死。

## 另一個沒想到的問題

**vCenter Server 要最後 suspend/shutdown。**

整個關機流程靠 vCenter API 呼叫。如果 vCenter 先關了，後面的 `suspend_vms()` / `enter_maintenance()` 全部失敗。

`priority_vms` 清單的最後一個永遠是 `vCenter Server`。
