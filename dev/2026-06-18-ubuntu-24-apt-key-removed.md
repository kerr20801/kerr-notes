# Ubuntu 24.04 把 apt-key 刪了

> **結論先說**：Ubuntu 22.04 開始 `apt-key` 已棄用，24.04 直接移除。舊腳本跑 `apt-key adv` 或 `apt-key add` 會直接 `command not found`。正確做法是用 `gpg --dearmor` 把金鑰存進 `/etc/apt/keyrings/`，然後在 source 定義裡指向該金鑰。

---

## 舊做法為什麼被砍

舊做法：

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```

問題是 `apt-key add` 把所有金鑰塞進一個全域 keyring（`/etc/apt/trusted.gpg`），裡面所有金鑰都信任**所有 repo**。意思是，如果你 A 廠商的 repo 金鑰被偷，它有能力簽署 B 廠商 repo 的套件，APT 照單全收。

Ubuntu 的解法是：每個 repo 用自己的金鑰檔案，source 定義裡明確指向對應金鑰，範圍完全隔離。

---

## Ubuntu 24.04 的正確做法

### 加入新 repo（以 Docker 為例）

```bash
# 1. 下載 GPG 公鑰，轉成 binary 格式
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
chmod a+r /etc/apt/keyrings/docker.gpg

# 2. 加入 source，指向剛才的金鑰
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
  | tee /etc/apt/sources.list.d/docker.list

apt update
```

關鍵差異：`signed-by=/etc/apt/keyrings/docker.gpg` — 這個 source 只接受被該特定金鑰簽署的套件。

---

## Ubuntu 24.04 的 DEB822 格式

Ubuntu 24.04 還引入另一個改變：預設 source 格式從 one-liner 改成 DEB822 多行格式。

舊格式（`/etc/apt/sources.list`）：

```
deb https://tw.archive.ubuntu.com/ubuntu noble main restricted universe multiverse
deb-src https://tw.archive.ubuntu.com/ubuntu noble main restricted universe multiverse
```

新格式（`/etc/apt/sources.list.d/ubuntu.sources`）：

```yaml
Types: deb deb-src
URIs: https://tw.archive.ubuntu.com/ubuntu
Suites: noble noble-updates noble-security
Components: main restricted universe multiverse
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg
```

兩種格式 24.04 都支援，但 Ubuntu 的預設 source 已經是 DEB822。如果腳本直接寫 `/etc/apt/sources.list`，24.04 上不會壞，但會有兩份 source 定義同時存在，`apt update` 時有時會出現重複 source 的 warning。

**偵測方式**：

```bash
if [[ -f /etc/apt/sources.list.d/ubuntu.sources ]]; then
    echo "DEB822 format"
else
    echo "Classic format"
fi
```

---

## 常見問題：NO_PUBKEY 錯誤

`apt update` 有時出現：

```
W: An error occurred during the signature verification. The repository is not updated
   and the previous index files will be used. GPG error:
   https://some.repo.com ... NO_PUBKEY A1B2C3D4E5F60001
```

這是 repo 更換金鑰，但你本機沒有新金鑰。修法：

```bash
# 從 Ubuntu keyserver 拉金鑰，存到 trusted.gpg.d
KEYID="A1B2C3D4E5F60001"
gpg --keyserver keyserver.ubuntu.com --recv-keys "$KEYID"
gpg --export "$KEYID" | tee /etc/apt/trusted.gpg.d/auto-fixed-${KEYID}.gpg > /dev/null
```

注意：`/etc/apt/trusted.gpg.d/` 這裡放的是仍然全域信任的金鑰（任何 source 都接受），但至少是以獨立檔案的形式，不是全部擠在一個 binary blob 裡。如果是自己的 repo 出 NO_PUBKEY，最好還是走 `signed-by` 的做法隔離。

---

## 升級腳本的 checklist

舊腳本遷移到 24.04 需要確認的事項：

| 舊做法 | 新做法 |
|--------|--------|
| `apt-key add -` | `gpg --dearmor -o /etc/apt/keyrings/<name>.gpg` |
| `apt-key adv --keyserver` | `gpg --keyserver ... --recv-keys` + export |
| source 不指定金鑰 | `signed-by=/etc/apt/keyrings/<name>.gpg` |
| `/etc/apt/sources.list` | 第三方 repo 用 `.list.d/` 或 DEB822 `.sources` |
| `apt-key list` | `ls /etc/apt/keyrings/ && gpg --show-keys /etc/apt/keyrings/<name>.gpg` |

**版本偵測**（讓一個腳本同時支援 22.04 和 24.04）：

```bash
UBUNTU_VERSION=$(. /etc/os-release && echo "$VERSION_ID")

if [[ "$UBUNTU_VERSION" > "23" ]]; then
    # Ubuntu 24.04+，使用 keyrings 方式
    KEYRING_DIR="/etc/apt/keyrings"
    mkdir -p "$KEYRING_DIR"
else
    # Ubuntu 22.04 以前，兼容舊方式（仍然建議改用 keyrings）
    KEYRING_DIR="/etc/apt/trusted.gpg.d"
fi
```

---

## 實際案例：GitLab runner repo

舊腳本：

```bash
curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh | bash
# 裡面用了 apt-key，24.04 直接炸
```

新做法：

```bash
curl -fsSL "https://packages.gitlab.com/runner/gitlab-runner/gpgkey" \
  | gpg --dearmor -o /etc/apt/keyrings/gitlab-runner.gpg

echo "deb [signed-by=/etc/apt/keyrings/gitlab-runner.gpg] \
  https://packages.gitlab.com/runner/gitlab-runner/ubuntu/ \
  $(lsb_release -cs) main" \
  | tee /etc/apt/sources.list.d/gitlab-runner.list

apt update && apt install gitlab-runner
```

不用依賴 GitLab 提供的安裝腳本，自己控制金鑰的存放位置。
