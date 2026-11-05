# 測試機指令集中稽核實戰筆記：用 Wazuh 做到「連清 history 都留痕」

> 本文整理自一位資安工程師朋友 Eason 的實戰筆記與設計決策記錄，內容已去除所有內部識別資訊（真實 IP/主機名一律以 `<PLACEHOLDER>` 表示）。感謝 Eason 授權分享。

## 背景 / 目的

多台測試機（Ubuntu 24.04 / 22.04 / Kali）上，任何使用者執行的指令都需要集中送到中央伺服器，供資安稽核與鑑識使用。核心要求只有一句話：**完整、難以繞過**——換 shell、清 history、在 script 裡間接呼叫，這些都不能讓紀錄消失。

## 設計決策：為什麼選這條路

這份筆記最有價值的地方不是安裝步驟，是每個技術選擇背後「為什麼」：

| 議題 | 決定 | 理由 |
|------|------|------|
| 記錄層級 | **kernel 層 `execve`/`execveat`**（透過 auditd） | shell history 可以被繞過（清掉、換 shell、script 內部呼叫都躲得掉）；execve 是鑑識級標準做法，任何方式執行程式都逃不掉 |
| 後端平台 | Wazuh 全家桶（Manager + Indexer + Dashboard） | 單一 agent、內建 audit 解析與告警規則、附帶 FIM/SCA/rootkit 偵測，不用自己刻查詢邏輯 |
| Server 部署 | 官方 `wazuh-install.sh` all-in-one 原生安裝（v4.14） | 一台主機即可，systemd 管理，5-15 分鐘裝完 |
| 端點佈署 | 手動安裝，照手冊逐步操作 | 台數少時沒必要先寫自動化，YAGNI |
| VM Clone | Golden image 模式：template 裝好 agent 但清除身分，clone 後重新註冊拿新 ID | 直接 clone 已註冊機器會讓多台共用同一個 agent ID，事件會混在一起、計數器錯亂 |

## 架構

```
測試機群（Ubuntu 24.04 / 22.04 / Kali）        Wazuh Server（Ubuntu，原生安裝）
┌─────────────────────────────┐         ┌───────────────────────────────────┐
│  auditd（execve 規則）       │ 1514/tcp│  wazuh-install.sh all-in-one      │
│    └► /var/log/audit/       ├────────►│  ├─ Wazuh Manager（解析/規則/告警）│
│  Wazuh Agent（讀取+上送）    │ 1515/tcp│  ├─ Wazuh Indexer（儲存/搜尋）    │
│                             │ (註冊)  │  └─ Wazuh Dashboard(443, Web UI)  │
└─────────────────────────────┘         └───────────────────────────────────┘
```

- 指令事件在 Dashboard 以 `rule.id:80792`（內建規則「Audit: Command」）查詢，不用自己寫解析規則。
- 記錄 `auid`（login UID）：即使 `sudo su -` 切換身分，仍可追溯到最初登入者。
- Manager 斷線期間 auditd 持續寫本機檔，恢復連線後 agent 自動補送（時間戳仍是實際執行時間）。

## Server 安裝

```bash
sudo apt update && sudo apt -y upgrade
sudo timedatectl set-ntp true          # NTP 必開，鑑識時間軸的前提
timedatectl status                     # 確認 "NTP service: active"

# 若有開 ufw，先放行必要 port
sudo ufw allow 1514/tcp   # agent 事件傳輸
sudo ufw allow 1515/tcp   # agent 註冊
sudo ufw allow 443/tcp    # Dashboard Web UI

curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
sudo bash ./wazuh-install.sh -a    # -a = all-in-one：Manager + Indexer + Dashboard
```

**結束時畫面會顯示 `admin` 的初始密碼，立刻抄下來**。沒抄到可從安裝產出的壓縮檔取回：

```bash
sudo tar -O -xvf wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt
```

裝完瀏覽器開 `https://<SERVER_IP>`（自簽憑證警告屬正常），確認三個服務都是 active：

```bash
systemctl status wazuh-manager wazuh-indexer wazuh-dashboard
```

### 開啟 enrollment force（支援機器重建/clone 頂掉舊註冊）

編輯 `/var/ossec/etc/ossec.conf`，`<auth>` 區段加入：

```xml
<auth>
  <force>
    <enabled>yes</enabled>
    <key_mismatch>yes</key_mismatch>
    <disconnected_time enabled="yes">1h</disconnected_time>
    <after_registration_time>1h</after_registration_time>
  </force>
</auth>
```

```bash
sudo systemctl restart wazuh-manager
```

意義：同名 agent 重新註冊時（例如測試機砍掉重 clone），Manager 自動汰換舊金鑰，不必手動介入。

## 測試機端安裝（Ubuntu / Kali 通用）

```bash
sudo -i    # 以下全部以 root 執行

timedatectl set-ntp true
timedatectl status

# 加入 Wazuh APT repo（Ubuntu、Kali 用同一個 Debian 系 repo）
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH \
  | gpg --dearmor -o /usr/share/keyrings/wazuh.gpg
chmod 644 /usr/share/keyrings/wazuh.gpg
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" \
  > /etc/apt/sources.list.d/wazuh.list
apt-get update
```

> **Kali 備註**：Kali 是 rolling release（Debian testing 基底），Wazuh 官方雖只列 Debian/Ubuntu，實測這個 repo 的套件在 Kali 上可正常安裝運作，不需特殊處理。

```bash
# 裝之前先把 hostname 改成你要的名字（agent 名稱預設 = hostname）
WAZUH_MANAGER="<MANAGER_IP>" apt-get install -y wazuh-agent
systemctl daemon-reload
systemctl enable --now wazuh-agent
```

安裝 auditd 並佈置 execve 規則：

```bash
apt-get install -y auditd
```

建立 `/etc/audit/rules.d/50-exec-log.rules`：

```
# 記錄所有「有登入身分」的使用者執行的程式（execve / execveat），64/32 位元都抓。
# key 使用 audit-wazuh-c：Wazuh 內建規則 80792 會直接解析為「Audit: Command」告警。
#
# auid!=unset：排除無登入身分的系統 daemon（開機服務等），大幅降噪；
#              所有互動登入的使用者（含 root、含 sudo su 之後）仍全數記錄。

-a always,exit -F arch=b64 -S execve,execveat -F auid!=unset -k audit-wazuh-c
-a always,exit -F arch=b32 -S execve,execveat -F auid!=unset -k audit-wazuh-c

# （選用）鑑識強化：取消下行註解後，規則將鎖定為 immutable，
# 任何人（含 root）都無法在不重開機的情況下卸載規則。
# 注意：之後要改規則必須重開機，測試期建議先不開。
# -e 2
```

```bash
augenrules --load
auditctl -l    # 應看到兩條 execve,execveat 規則
```

確認 `/var/ossec/etc/ossec.conf` 有讀取 audit log（Linux 預設通常已內建）：

```xml
<localfile>
  <log_format>audit</log_format>
  <location>/var/log/audit/audit.log</location>
</localfile>
```

有改動要重啟 agent：`systemctl restart wazuh-agent`

### 煙霧測試

```bash
whoami && id
```

到 Dashboard → Threat Hunting（或 Discover），過濾 `rule.id:80792`，一分鐘內應看到「Audit: Command」事件，可展開 `data.audit.command`、`data.audit.execve.a0/a1/...`（完整參數）、`data.audit.auid`（登入者 UID）。

## 排錯：「sudo 的指令有錄到，一般使用者的沒有」

這個症狀幾乎都是 **audit 管線沒通**：看到的 `sudo` 事件其實是 auth.log 那條線的 rule 5402（Successful sudo），不是 audit 的 80792。點開事件看 `rule.id` 即可證實。

**第 1 關：auditd 本身有沒有記？**

```bash
id                                              # 用一般使用者跑一條指令
sudo ausearch -k audit-wazuh-c -i | tail -30    # 查得到剛剛的 id 才算通
```

`<no matches>` → 規則沒載入，依序檢查 `auditctl -l`、確認 `50-exec-log.rules` 存在、重新 `augenrules --load`、確認 `auditd` 是 active。

**第 2 關：agent 有沒有收 audit.log？**（最容易漏）

```bash
grep -n -A2 'audit.log' /var/ossec/etc/ossec.conf   # 要有 <localfile> audit 區塊
sudo systemctl restart wazuh-agent                   # 改過設定要重啟
sudo grep "audit.log" /var/ossec/logs/ossec.log | tail -5
# 應看到：INFO: Analyzing file: '/var/log/audit/audit.log'
```

**第 3 關（少見）：Manager 端 key 對照表**

```bash
sudo cat /var/ossec/etc/lists/audit-keys   # 要有 audit-wazuh-c:command（預設就有）
```

三關都通後，一般使用者跑 `whoami`，一分鐘內 Dashboard `rule.id:80792` 就會出現。

有一個常踩的坑值得特別記下來：如果不是以 `sudo -i` 操作，而是用 `WAZUH_MANAGER=... sudo apt install ...`，**`sudo` 會把環境變數清掉**，設定檔裡會留下佔位字串 `MANAGER_IP`，agent 起不來（錯誤：`Invalid server address found: 'MANAGER_IP'`）。正確寫法是把變數放在 `sudo` 之後：`sudo WAZUH_MANAGER="<IP>" apt-get install -y wazuh-agent`。已經裝下去的話不用重裝，直接改設定重啟即可：

```bash
sudo sed -i 's/MANAGER_IP/<實際IP>/' /var/ossec/etc/ossec.conf
sudo systemctl restart wazuh-agent
```

## VM Clone / Golden Image 流程

### 為什麼不能直接 clone 已註冊的機器

Wazuh agent 的身分是 `/var/ossec/etc/client.keys` 裡的 **agent ID + 金鑰**，不是 hostname。直接 clone 會讓多台機器共用同一個 agent ID：所有事件混在同一個身分下分不出來源機器，事件計數器（anti-replay counter）各自累加造成錯亂，Manager 拒收、agent 反覆斷線。

正確做法：template 裡「裝好但不帶身分」，clone 出來的機器各自重新註冊拿新 ID。

**Template 封裝前**（在 template 機上執行，然後關機）：

```bash
sudo -i

# 1. 停 agent、清身分金鑰
systemctl stop wazuh-agent
systemctl disable wazuh-agent          # 防止 clone 開機時搶在改 hostname 前註冊
rm -f /var/ossec/etc/client.keys

# 2. 清 agent 本機事件計數狀態
rm -rf /var/ossec/queue/rids/*

# 3. 清 machine-id（clone 的標準衛生，避免 DHCP/journal 身分重複）
truncate -s 0 /etc/machine-id
rm -f /var/lib/dbus/machine-id
ln -s /etc/machine-id /var/lib/dbus/machine-id

# 4. 關機，拿去做 template
poweroff
```

**每台 Clone 開機後**（新機器上執行）：

```bash
sudo -i
hostnamectl set-hostname <新機器名稱>   # 之後 Dashboard 上看到的 agent 名稱
timedatectl set-ntp true
systemctl enable --now wazuh-agent      # 自動向 Manager 註冊、取得全新 agent ID

grep ^status /var/ossec/var/run/wazuh-agentd.state   # status='connected'
cat /var/ossec/etc/client.keys                        # 新的 ID + 新 hostname
```

auditd 規則已包在 template 裡，clone 後不需重做。

| 常見症狀 | 原因 / 處理 |
|------|-------------|
| clone 開機後 Dashboard 出現的 agent 名稱是 template 的舊 hostname | agent 在改 hostname 前就註冊了。停 agent、刪 `client.keys`、改好 hostname 再啟動一次 |
| 同名機器砍掉重 clone 後註冊失敗 | Manager 端 `<auth><force>` 未開啟 |
| 兩台機器在 Dashboard 上輪流上下線 | 兩台共用了同一份 `client.keys`（clone 前沒清），擇一重新清理註冊 |

## 驗收清單

每台測試機裝完後逐項執行，全部打勾才算部署完成：

**基本連通**
- [ ] Dashboard → Agents：該機器狀態 Active
- [ ] `grep ^status /var/ossec/var/run/wazuh-agentd.state` 為 `connected`
- [ ] `auditctl -l` 有兩條 `execve,execveat` 規則
- [ ] NTP active，事件時間戳與實際時間一致

**指令記錄完整性（鑑識重點：每種情境都要在 Dashboard 查得到）**

| # | 情境 | 測試指令 | 預期結果 |
|---|------|----------|------------------|
| 1 | 一般使用者 | `whoami` | `auid` = 該使用者 UID |
| 2 | sudo 執行 | `sudo cat /etc/shadow` | sudo 與 cat 兩筆都記錄，`auid` 仍是原登入者 |
| 3 | sudo su 切 root | `sudo su -` → `ls /root` | `uid=0` 但 `auid` 仍是原登入者（可追溯） |
| 4 | 換 shell | `zsh` 進去執行 `id` | 一樣被記錄（execve 層級不分 shell） |
| 5 | 間接執行 | `python3 -c "import os; os.system('uname -a')"` | python3 與 uname 都留下紀錄 |
| 6 | 清除痕跡 | `history -c && rm ~/.bash_history` | 前面事件仍在 Server 上，`rm` 本身也被記錄 |

**斷線韌性**
- [ ] 暫時擋住測試機到 Server 的 1514/tcp，期間執行幾條指令
- [ ] 恢復連線後幾分鐘內，斷線期間的指令補送到 Dashboard（時間戳為實際執行時間）

**Clone 流程**
- [ ] 同一 template clone 出的兩台機器，在 Agents 頁是兩個不同 agent ID
- [ ] 各自執行測試指令，事件正確歸屬到各自機器名下

## Dashboard 查詢與容量維運

固定過濾條件 `rule.id:80792`（Audit: Command），把 SCA、sudo auth log 等雜訊濾掉。建議選取的欄位：

| 欄位 | 意義 |
|------|------|
| `timestamp` | 執行時間 |
| `agent.name` | 哪台測試機 |
| `data.audit.auid` | **原始登入者 UID**（鑑識關鍵：`sudo su` 之後也不變；1000=第一個一般帳號、0=直接 root 登入） |
| `data.audit.uid` / `euid` | 執行當下的身分（與 auid 對照可看出有無提權） |
| `data.audit.exe` | 執行的 binary 完整路徑 |
| `data.audit.execve.a0` ~ `aN` | 完整指令與參數（a0=指令，之後是參數） |
| `data.audit.cwd` | 執行時所在目錄 |
| `data.audit.tty` | 來源終端（同一 tty ≈ 同一工作階段，可把指令串成故事線） |

排好後在 Discover 右上 Save 存成「指令稽核時間軸」，之後一鍵開啟。

### 容易誤會的欄位

- `data.audit.command` = kernel 的 `comm`（程式名，最長 16 字元、**不含參數**）；看完整指令認 `execve.a0~aN`。
- `data.audit.auid` 是數字 UID，不是帳號名；`getent passwd <UID>` 對照。
- `data.sca.check.command` **不是使用者下的指令**——那是 SCA（基線檢查）模組自己定期執行的檢查指令，屬於系統體檢，與使用者行為無關：

| | 過濾條件 | 關鍵欄位 |
|---|----------|----------|
| 使用者指令稽核 | `rule.id:80792` 或 `rule.groups:audit` | `data.audit.*` |
| SCA 基線檢查（系統自跑） | `rule.groups:sca` | `data.sca.check.*` |
| sudo 認證記錄（另一條線） | `rule.id:5402` | `data.srcuser` 等 |

> `sudo` 指令會同時觸發 5402（auth.log）與 80792（audit）兩種事件。只看到 5402 而沒有 80792，代表 audit 管線沒通。

### 容量規劃

一筆 execve 告警在 Indexer 約佔 1-2 KB（含索引開銷）：

| 使用型態 | 每台每日事件量 | 5 台每日磁碟 | 200 GB 可放 |
|----------|----------------|--------------|-------------|
| 人工互動操作 | 數千～2 萬筆 | ~50-150 MB | 好幾年 |
| 自動化測試/編譯 | 10 萬+ 筆 | ~1 GB | 約半年 |
| 極端（fuzzing、大量掃描） | 百萬筆級 | 5-10 GB+ | 1-2 個月 |

裝完跑幾天後量真實用量（Dashboard → Dev Tools）：

```
GET _cat/indices/wazuh-alerts-*?v&h=index,docs.count,store.size&s=index
```

**自動刪舊索引（建議必設）**：Dashboard → Index Management → State management policies，建 ISM policy 讓 `wazuh-alerts-*` 超過 90 天（依鑑識保留需求調整）自動刪除，磁碟用量會在固定水位飽和，不會塞滿。

**某台機器特別吵的時候**（跑大量自動化/fuzzing 導致事件量暴增百倍）：在那台的 audit 規則加排除即可，整體架構不動——在 `50-exec-log.rules` 的 `-a always,exit` 兩行**之前**加：

```
# 排除跑自動化的帳號（例：UID 1001）
-a never,exit -F arch=b64 -S execve,execveat -F auid=1001
-a never,exit -F arch=b32 -S execve,execveat -F auid=1001
```

改完 `sudo augenrules --load` 生效。

## 小結

這套設計的核心價值不在「裝了一個 SIEM」，而在幾個決策點：**選 kernel 層而不是 shell history**（繞不過去）、**用 auid 而不是 uid 追人**（sudo su 之後還認得出來）、**clone 前一定要清身分**（不然事件全部混在一起）。這三點想清楚了，剩下都只是照官方文件裝軟體。

---

*再次感謝 Eason 授權整理分享這份設計筆記與實戰紀錄。*
