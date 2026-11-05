# NAS AI：八段式檔案安全 Pipeline 設計

> **結論先說**：檔案安全掃描的核心問題不是「用哪個掃毒引擎」，而是「用對掃描順序」。貴的操作放最後，0ms 能擋掉的放最前面。Isolation Forest 只學乾淨樣本，這一條讓整個 ML 模型防投毒。DLP 放最後，因為它只關心內容語意，不管檔案是否可執行。

---

## 架構概覽

```
任何設備上傳
    ↓
Stage 0  SHA256 hash 黑名單（已知惡意 → 毫秒擋掉）
Stage 1  副檔名黑名單（21 種可執行檔）
Stage 2  MIME 偵測 + ext/declared 交叉驗證
Stage 7  Zip-bomb guard（只讀中央目錄，不解壓）
Stage 3  Entropy 計算（壓縮格式自動跳過，避免誤判）
Stage 4  Isolation Forest（只學 clean 樣本，防投毒）
Stage 5  ClamAV INSTREAM（strict profile 才跑）
Stage 6  DLP（credentials + 台灣 PII，password regex 已收斂）
    ↓                           ↓
  路由到 NAS          quarantine + TG 通知
```

注意 Stage 7 在 3、4、5 之前。Zip-bomb guard 必須在任何試圖「讀取更多內容」的 stage 之前。

---

## 為什麼要分 Profile

不是每個上傳場景都需要跑 ClamAV。ClamAV INSTREAM 是最慢的 stage，每次至少幾百毫秒，掃大檔案會超過1秒。

| Profile | Stage | 適用場景 |
|---------|-------|---------|
| fast | 0,1,2 | 高頻低風險（員工截圖、文件） |
| standard | +3,4,6,7 | 一般上傳（預設） |
| strict | +5 | 外部收件、客戶上傳 |
| archive | 0,1,2,3,7 | 壓縮包（不跑 ML，跑 entropy） |

設計原則：Profile 在 config 定義，不動 code：

```yaml
targets:
  home:
    path: /mnt/nas-home/incoming
    profile: standard
  company:
    path: /mnt/nas-company/uploads
    profile: strict
    allowed_types: ["pdf", "docx", "xlsx"]
```

加 NAS 只改 YAML。

---

## Stage 順序的邏輯

**最快的先跑**。Hash 查表是 O(1)，幾乎 0ms。副檔名 string match 也是 0ms。MIME 偵測需要讀幾百 bytes，快。Entropy 需要讀整個檔案，慢。ClamAV 需要傳整個檔案給 daemon 掃，最慢。

Stage 0 的 hash 黑名單：

```python
BLOCKLIST = {
    "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",  # 空檔案
    "d41d8cd98f00b204e9800998ecf8427e",  # 已知 malware hash
    # ...
}

def stage0_hash_blocklist(sha256: str) -> bool:
    return sha256 in BLOCKLIST
```

上傳時先算 SHA256，hash 在黑名單直接擋，不做任何後續處理。這個 stage 也讓「已知惡意」的變種完全繞不過去，無論他怎麼改名或壓縮。

---

## MIME 交叉驗證

只看副檔名不夠，`.jpg` 裡面可能是 PE 執行檔。只看 MIME 也不夠，`.exe` 改名成 `.jpg` 的 MIME 偵測會回 `image/jpeg`（如果有偽造的 JFIF header）。

Stage 2 同時做三件事：

1. `libmagic` 偵測實際 MIME
2. 比對 declared（副檔名暗示的）MIME
3. 比對 content-type header（客戶端宣稱的）

任何一組矛盾就標成 suspicious：

```python
if actual_mime != expected_mime_for_ext:
    result = "suspicious"
    reason = f"MIME mismatch: declared={ext}, actual={actual_mime}"
```

---

## Zip-bomb Guard：只讀中央目錄

Zip-bomb 的原理：壓縮率超高的 zip 檔，解壓後可能從幾KB變成幾十GB，撐爆磁碟或記憶體。

防禦關鍵：**不解壓，只讀中央目錄（Central Directory）**。ZIP 格式的中央目錄在檔案結尾，記錄了每個檔案的壓縮前/後大小。

```python
def stage7_zipbomb_guard(file_path: str) -> bool:
    with zipfile.ZipFile(file_path) as zf:
        total_uncompressed = sum(info.file_size for info in zf.infolist())
        total_compressed = sum(info.compress_size for info in zf.infolist())

        if total_uncompressed > 1_073_741_824:  # 1GB
            return False  # 太大
        if total_compressed > 0 and (total_uncompressed / total_compressed) > 100:
            return False  # 壓縮比超過 100x
    return True
```

整個過程只讀中央目錄元數據，不解壓任何內容。

---

## Isolation Forest 為什麼只學 clean 樣本

Isolation Forest 是 anomaly detection 演算法，不是分類器。它的邏輯是：正常樣本在特徵空間裡聚集，異常樣本是孤立點，容易被「隔離」。

**反投毒設計**：如果讓 IF 也學惡意樣本，攻擊者可以分批上傳惡意檔案，讓 IF 把惡意特徵學成「正常」，之後類似的惡意檔案就繞過去了。

只學 clean 樣本，IF 只知道什麼是「正常」，任何偏離正常特徵的都當 anomaly。攻擊者無法讓 IF 學到惡意特徵。

特徵設計：

```python
features = [
    file_size,
    entropy,
    has_pe_header,  # 0 or 1
    ratio_printable_chars,
    ratio_null_bytes,
    unique_byte_values,
]
```

Lazy refit：不是每次上傳都重新訓練，而是累積一定量的 clean 樣本後才 refit，避免 CPU 尖峰。

---

## Entropy Stage 的壓縮格式跳過

Entropy（Shannon entropy）是衡量資訊密度的指標。加密或混淆的程式碼 entropy 接近 8.0（每byte資訊量最大）。未混淆的文字檔 entropy 通常 4.0~5.5。

**問題**：`.zip`、`.jpg`、`.gz` 等已壓縮格式本身 entropy 就接近 8.0，這不代表裡面有惡意內容。

```python
COMPRESSED_TYPES = {
    "application/zip",
    "application/gzip",
    "image/jpeg",
    "image/png",
    # ...
}

def stage3_entropy(actual_mime: str, file_path: str) -> bool:
    if actual_mime in COMPRESSED_TYPES:
        return True  # 跳過，不計 entropy

    entropy = calculate_shannon_entropy(file_path)
    return entropy < ENTROPY_THRESHOLD  # 閾值通常 7.2~7.5
```

不跳過的話，所有圖片和壓縮檔都會被誤判為高 entropy 可疑。

---

## DLP：不只是 regex

Stage 6 偵測兩類東西：

**Credentials（通用）**：
- AWS key pattern：`AKIA[0-9A-Z]{16}`
- Private key header：`-----BEGIN RSA PRIVATE KEY-----`
- Generic password patterns（已收斂，避免誤判）
- JWT token format

**台灣 PII**：
- 身分證字號：`[A-Z][12][0-9]{8}`
- 統一編號：8位，符合加權驗證
- 手機：`09[0-9]{8}`
- 信用卡：Luhn check

Password regex 收斂很重要。早期版本用 `password\s*[=:]\s*\S+` 這樣的 pattern，`mysql -u root -ppassword` 這行指令也會觸發。現在的版本需要有 context（比如在設定檔格式的行）才觸發，降低誤報。

---

## Routing：Clean vs Quarantine

這條是最重要的設計決策：**只有通過全部 stage 的才進 NAS，suspicious/malicious 永遠進 quarantine**，不會繞路到 NAS。

之前有個 routing bug：suspicious 檔案在某些 profile 下還是會被路由到 NAS（因為 profile 沒有跑到 Stage 6，就當 clean 處理）。修法是把 routing 決策從 profile 裡抽出來，分析結果只要有任何 stage 標為 suspicious，就進 quarantine，與 profile 無關。

```python
def route_file(analysis_result: AnalysisResult, target: str) -> str:
    if analysis_result.verdict in ("suspicious", "malicious"):
        return QUARANTINE_PATH  # 永遠 quarantine
    return NAS_TARGET_PATHS[target]
```

Quarantine 後發 TG 通知，讓管理員看到是哪個 stage 擋住的、原始 SHA256、上傳來源。
