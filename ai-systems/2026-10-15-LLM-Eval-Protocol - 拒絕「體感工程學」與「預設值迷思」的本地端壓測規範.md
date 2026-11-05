LLM-Eval-Protocol: 拒絕「體感工程學」與「預設值迷思」的本地端壓測規範
這套規範的誕生，來自於無數小時在本地端（如 ASUS GX10、Mac Mini 等資源受限環境）除錯與調校的血淚教訓。在本地端跑 LLM 沒有魔法，只有硬梆梆的物理極限跟記憶體頻寬。

我已經受夠了每次開會或看社群討論時，都要聽別人通靈說「我覺得 vLLM 比較順」、「那個誰說 Qwen 小模型有 80% Opus 的水準」。為了停止這種毫無數據支撐的口水戰，本 Repo 定義了一套極度嚴格的「可重現比較協議 (Reproducible Comparison Protocol)」。

以後要討論哪個框架快、哪個模型聰明，請直接帶上符合本規範的 Log 發 PR，沒有數據的體感玄學一律不予回應。

⚠️ 最大的迷思：把「不會調參」當成「框架效能差異」

社群裡最常聽到的都市傳說就是：「同一個模型，vLLM 跑起來就是比 Ollama 快很多，Ollama 效能不行。」

說穿了，這通常只是「預設值差異」與「使用者未做調優 (Tuning)」的結果。在相同的硬體極限與記憶體頻寬下（例如 273 GB/s 的機器跑 70B 模型，單人吐字就是鎖死在物理極限的 6 t/s），底層的矩陣運算量是固定的。vLLM 預設可能激進地吃滿 VRAM 劃分 KV Cache 以優化吞吐量；而 Ollama 為了相容廣大開發者的設備，預設參數給得相對保守。

如果你不懂得依照硬體去修改環境變數或 Modelfile，直接拿兩邊的「預設值」來硬比，然後得出「某某框架效能比較差」的結論，這叫技術懶惰，不叫壓測。

壓測鐵律 (The Protocol)

如果要在本專案/團隊內提交任何模型或框架的效能對比，必須 100% 遵守以下規則：

1. 拒絕玩具環境，上正式 API 並公開參數配置
別拿網頁截圖或第三方平台的 Demo 畫面來吵。要測就是起兩個 Docker Container，直接對接你預計上線的 Production Endpoint（不管是 Ollama, vLLM 還是 SGLang），並且必須明確列出雙方的 Configuration。不要拿 tuning 過的 A 去打預設值的 B。

2. 嚴格變因控制：雙開 Session 同步壓測
不要今天測 A，明天測 B，然後跟我說 A 比較快。請寫個簡單的 Python 腳本（參考 scripts/dual_shootout.py），使用 asyncio 在同一秒鐘對兩個 Endpoint 發送完全一樣的 Prompt，並精確記錄首字延遲 (TTFT) 與每秒生成速度 (Tok/s)。

3. 實戰考題（禁止 Hello World）
如果你要把 LLM 放到正式的 AIOps 或 SRE 監控管線裡，我們需要的是「穩定輸出巢狀 JSON 不崩潰的能力」，而不是「寫請假信寫得很通順」。
本 Repo 的 test_prompts/ 裡面放的是真實且充滿雜訊的 Zabbix 告警紀錄跟 NGINX Error Log。測試任務很簡單：

找出異常 IP 跟發生時間。

嚴格輸出 JSON 格式，多一個 Markdown 標記或廢話就視為 Fail。

連跑 5 次，看哪個模型先精神分裂。

4. 留下 Log 呈堂證供
「感覺比較好」不是一個有效的 Metrics。提交測試結果時，必須附上完整的輸入/輸出 Log、時間戳記與硬體規格（VRAM 佔用、CPU 負載、記憶體頻寬）。讓任何人拉下這個 Repo，執行 make test 都能得到跟你一模一樣的結論。重現不出來的，就是運氣，不是效能。

Quick Start (如何跑這個壓測腳本)

別廢話了，自己跑一次就知道。

```
# 1. Clone repo
git clone https://github.com/yourname/Anti-BS-LLM-Eval.git
cd Anti-BS-LLM-Eval

# 2. 設定你要對打的兩個 Endpoint
export LLM_A_URL="http://127.0.0.1:11434/v1/chat/completions" # 例如 Ollama
export LLM_B_URL="http://127.0.0.1:8000/v1/chat/completions"  # 例如 vLLM

# 3. 開揍 (預設會跑 5 輪 Zabbix Log 分析任務)
python3 scripts/run_benchmark.py --payload tests/zabbix_hardcore.json
```
