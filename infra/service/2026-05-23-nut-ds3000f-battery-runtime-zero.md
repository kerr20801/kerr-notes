# NUT DS3000F：battery.runtime 永遠回傳 0

## 問題

用 NUT `nutdrv_qx` 管理 DS3000F UPS（6×12V7Ah），跑 `upsc optiups@localhost` 看到：

```
battery.charge: 87
battery.runtime: 0
ups.status: OB
```

`battery.runtime` 在市電正常時是 0，斷電後還是 0。這台 UPS 的 driver 不計算剩餘時間，永遠回傳 0。

## 解法：從電量% 反推

用物理推算代替 SNMP 回傳值：

```
可用電量（Wh） = 電池容量 × DoD × 效率
             = 6 × 12V × 7Ah × 80% × 85%
             ≈ 343 Wh

剩餘時間（min）= (battery.charge / 100) × (343 / 當前負載W) × 60
```

實作：

```python
if status.battery_remaining_sec == 0 and status.battery_charge_pct > 0:
    _BATT_WH  = 343.0
    _UPS_WATT = 2700.0
    load_w = _UPS_WATT * max(status.output_load_pct, 5) / 100
    max_runtime_min = _BATT_WH / load_w * 60
    status.battery_remaining_sec = int(
        status.battery_charge_pct / 100.0 * max_runtime_min * 60
    )
```

`max(load_pct, 5)` 是為了避免 load=0 時除以零，UPS 本身也有靜態功耗。

## 準確度

這個估算比實際偏高，SNMP 估算普遍高估 20% 左右。所以在狀態機裡用 `predict_runtime()` 再乘 0.8 修正：

```python
# 沒有訓練資料時的 fallback
return snmp_estimate_min * 0.8
```

累積幾次真實停電事件後，CatBoost 模型會取代這個估算，準確度更好。

## 確認你的 UPS 是否有同樣問題

```bash
upsc <ups-name>@localhost | grep battery.runtime
# 永遠是 0 → 用上面的方法
# 有正常數值 → 直接用，不需要估算
```
