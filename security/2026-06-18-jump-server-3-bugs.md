# Jump Server 三個靜默地雷

> **結論先說**：`chmod 666` 放在 server config 上，所有人都可以改 SSH 目標。curl 拼 JSON 字串，訊息裡一個換行就把 API call 炸掉。bash local 變數不繼承給 callee，用到空值不報錯，最難查。三個問題都不會即時報錯，會靜默爛掉。

---

## 地雷一：chmod 666 的 server list

Jump server 常見的共享設計：一個 `/etc/bastion_servers.conf` 記所有目標主機，所有用戶都可以連。

問題在這裡：

```bash
# 讓所有人可以讀寫
chmod 666 /etc/bastion_servers.conf
```

格式大概長這樣：

```
prod-db1|root|192.168.1.84
prod-api|deploy|192.168.1.85
```

**攻擊面**：任何有 shell 的用戶，一行指令把目標改掉：

```bash
echo "fake-prod|attacker|1.2.3.4" >> /etc/bastion_servers.conf
# 或直接覆寫
echo "prod-db1|root|1.2.3.4" > /etc/bastion_servers.conf
```

下次有人選「prod-db1」，SSH 就打到攻擊者的機器。這不是理論漏洞，是一行指令就能做的事。

**正確做法**：

```bash
# root 擁有，jump_users 群組可讀寫，其他人不行
chown root:jump_users /etc/bastion_servers.conf
chmod 664 /etc/bastion_servers.conf
```

用戶要能改 server list，就把他們加進 `jump_users` 群組，不是把 config 開給所有人。

---

## 地雷二：curl 拼 JSON 字串

多平台告警常見的寫法：

```bash
FORMATTED_MESSAGE="用戶: $USER\n來源: $SSH_IP\n狀態: 已登入"

# Mattermost
curl -s -X POST -H "Content-Type: application/json" \
  -d "{\"text\": \"$FORMATTED_MESSAGE\"}" \
  "$MATTERMOST_URL"

# Discord
curl -s -X POST -H "Content-Type: application/json" \
  -d "{\"content\": \"$FORMATTED_MESSAGE\"}" \
  "$DISCORD_URL"
```

**問題**：`$FORMATTED_MESSAGE` 裡的 `\n` 是字面 backslash-n，shell 不展開。真的有換行（比如來自 hostname 輸出或 log 內容），JSON 就非法了：

```json
{"text": "用戶: alice
來源: 192.168.1.100
狀態: 已登入"}
```

JSON 字串不允許未 escape 的換行，`curl` 這個 request 要不靜默失敗、要不 API 回 400。

**另一個問題**：如果訊息裡有雙引號（比如路徑名稱 `/home/user/"project"`），JSON 直接爛掉。

**正確做法**：用 `jq` 構建 JSON，它會處理所有 escaping：

```bash
payload=$(jq -n --arg text "$FORMATTED_MESSAGE" '{"text": $text}')
curl -s -X POST -H "Content-Type: application/json" \
  -d "$payload" "$MATTERMOST_URL"
```

`jq -n --arg` 會把 `$FORMATTED_MESSAGE` 裡的換行、引號、反斜線全部正確 escape，不用自己處理。

---

## 地雷三：bash local 變數不繼承 callee

這個最難查，因為完全不報錯。

```bash
create_user() {
    local role="admin"
    local auto_generate_key="true"

    # ... 建立用戶 ...

    configure_security "$new_user"   # ← 呼叫另一個 function
}

configure_security() {
    local username="$1"

    # 這裡用 $role 和 $auto_generate_key
    cat > "/home/$username/.config" <<EOF
ROLE="$role"
SSH_KEY="$auto_generate_key"
EOF
    # 結果：$role 和 $auto_generate_key 都是空字串
}
```

**行為**：`create_user` 裡的 `local role` 和 `local auto_generate_key` 在 `configure_security` 裡完全不可見。輸出的 config 長這樣：

```
ROLE=""
SSH_KEY=""
```

不報錯，靜默寫了空值。如果後續邏輯靠這個 config 做判斷，下游全部靜默跑錯。

**為什麼會誤以為可以用**：在同一個 function 裡 call 的 subshell（`$()`）和 pipe 確實拿不到，但直接呼叫另一個 function 有時可以拿到，這是因為 bash 有 dynamic scoping 的感覺，實際上只有 `local` 才是真正的 local，不加 `local` 的變數是全域的。

```bash
# 這樣可以
create_user() {
    role="admin"            # 不加 local，全域
    configure_security "$new_user"  # configure_security 看得到 $role
}

# 但這樣不行
create_user() {
    local role="admin"      # local，只活在 create_user
    configure_security "$new_user"  # configure_security 看不到 $role
}
```

**正確做法**：需要什麼就傳什麼，不靠隱式繼承：

```bash
create_user() {
    local role="admin"
    local auto_generate_key="true"
    configure_security "$new_user" "$role" "$auto_generate_key"
}

configure_security() {
    local username="$1"
    local role="$2"
    local auto_generate_key="$3"
    # 明確收到，不靠繼承
}
```

---

## 共同點

三個問題都有同一個特徵：**不會即時崩潰，會靜默產生錯誤結果**。

- `chmod 666`：系統照常運作，直到有人利用它
- curl 拼 JSON：告警看起來有發，API 悄悄丟掉了
- bash local：config 照常寫，值是空的

這類問題比直接崩潰更難抓，因為你不知道它已經壞了。
# 結合 fail2ban 驗證是不錯做法不過我沒去做
