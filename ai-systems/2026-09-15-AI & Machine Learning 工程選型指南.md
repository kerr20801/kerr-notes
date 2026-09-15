AI & Machine Learning 工程選型指南 (包含序列與變點分析)
第一部分：傳統機器學習 (Traditional ML)
💡 核心特徵： 處理「結構化表格數據（Logs、DB、統計指標）」的首選。推論極快（多數 CPU 即可跑）、具備高可解釋性，方便追溯審計。

1. 數值與趨勢預測 (Regression)
線性/多項式迴歸 (Linear / Polynomial Regression)： 預測資源消耗（如 Disk IO）的單純趨勢線。

正規化迴歸 (Ridge / Lasso)： 當特徵欄位極多且互相干擾時使用，Lasso 能自動幫你踢除沒用的特徵。

2. 基礎分類器 (Classification)
邏輯迴歸 (Logistic Regression)： 最基礎的二元分類（如：單一 API 請求是否異常）。

K-近鄰演算法 (KNN)： 靠「幾何距離」找相似度。適合基礎的異常行為比對。

單純貝氏 (Naive Bayes)： 基於機率論，適合處理惡意 Payload 的關鍵字特徵過濾。

支持向量機 (SVM)： 適合樣本數不多，但特徵空間複雜的精準邊界偵測。

3. 強大樹狀模型與集成 (Tree-based & Ensemble) —— 表格數據霸主
隨機森林 (Random Forest)： 綜合多棵決策樹，抗雜訊強，適合高穩定性的審核機制。

XGBoost / LightGBM： 處理結構化數據的王者。適合行為分類、惡意流量預測。LightGBM 在海量 Log 下速度極快。

CatBoost： 專門對付「文字類別標籤（Categorical）」極多的日誌資料，免手動轉換。

投票與堆疊 (Voting & Stacking)： 將多個模型綁在一起做最終決策，追求極致的準確與穩定度。

4. 無標籤找規律與關聯 (Unsupervised & Association)
K-Means / DBSCAN 分群： 將行為相似的 IP 聚類。DBSCAN 特別適合過濾空間或數值上的離群雜訊。

Apriori / FP-Growth： 不看特徵，只找「同時發生」的機率（例如：A 服務超時，通常伴隨 B 節點報錯的系統告警關聯分析）。

5. 異常與變點檢測 (Anomaly & Change Point Detection) 👉 [新增補齊]
z-score (標準分數) / 孤立森林 (Isolation Forest)： 抓出「單點的極端異常」（例如 CPU 瞬間飆高、WAF 上的突發惡意請求）。

變點偵測 (Change Point Detection, 如 PELT 演算法)：

解決什麼問題： 異常偵測是抓「突刺」，變點偵測是抓「基準線的永久偏移」。

何時用： 系統更新或 K8s 部署後，記憶體消耗是否產生了新的常態高原？網路路由更改後，延遲是否發生了結構性改變？

6. 狀態推論與序列標記 (State Inference & Sequence Labeling) 👉 [補回遺漏]
隱馬可夫模型 (HMM, Hidden Markov Model)：

解決什麼問題： 透過觀察到的表面行為，推測背後隱藏的「狀態」。

何時用： 從一系列的網路存取 Log 中，推論攻擊者目前處於哪個攻擊階段（如：掃描 -> 嘗試利用 -> 橫向移動）。

條件隨機場 (CRF, Conditional Random Field)：

解決什麼問題： 根據前後文脈絡，對序列中的每個元素打上標籤。

何時用： 實體識別（NER）與 Log 解析。例如將一長串無結構的 Raw Log，精準拆解出「IP」、「時間」、「執行動作」等欄位。CPU 執行極快且特徵權重完全可解釋。

7. 降維與時間序列 (Dimensionality & Time Series)
PCA / t-SNE / UMAP： 把幾百個欄位壓縮成 2D/3D，用來畫圖觀察資料分佈。

Prophet (Meta 開源)： 專門處理有「強烈週期性（日、週、月）與節假日效應」的伺服器連線數預測或資源擴容（Auto-scaling）評估。

第二部分：深度學習 (Deep Learning)
💡 核心特徵： 處理「非結構化數據（影像、聲音、長文本、拓撲圖）」的重型武器。仰賴 GPU，能自動萃取特徵。

人工神經網路 (MLP / ANN)： 模擬神經元互聯的基礎網路，處理高度非線性的特徵交叉。

卷積神經網路 (CNN)： 處理影像與空間特徵的霸主。

長短期記憶網路 (LSTM / GRU)： 帶有「記憶機制」，專解時間先後順序強烈相關的問題。

圖神經網路 (GNN)： 專門把「節點」與「連線」放進去運算。極度適合殭屍網路拓撲偵測、複雜資金/流量關聯圖譜。

Transformer 架構 (BERT, GPT)： 靠自注意力機制理解全局上下文，現代 NLP 核心。

嵌入模型 (Embedding)： 將文本轉為數學向量，專解「關鍵字不同但語意相近」的模糊搜尋。

自動編碼器 (Autoencoder)： 把正常資料「壓縮再還原」，用還原失敗（重建誤差）來抓出進階異常。

第三部分：強化學習 (Reinforcement Learning)
💡 核心特徵： 不給歷史答案，讓 AI 在環境中「試錯」，透過「獎懲機制」找出最佳策略。適合動態決策。

Q-Learning / DQN： 適合離散動作控制（如基礎機器人路徑規劃）。

PPO (Proximal Policy Optimization)： 主流的連續控制演算法。應用於 LLM 偏好對齊（RLHF）、自動駕駛，以及動態網路負載平衡策略。

第四部分：現代 AI 工程實務 (GenAI / LLMOps)
💡 核心特徵： 從頭練模型成本太高，現在專注於「微調、串接、部署與自動化工作流」。

RAG (檢索增強生成)： 結合 BM25 或向量庫，讓 LLM 根據企業私有文件回答，消除幻覺。

Agentic Workflow (Function Calling)： 賦予 LLM 呼叫外部 API 的能力，打造自動化應變體系。

PEFT / LoRA： 單張 GPU 即可進行的模型微調，讓開源模型快速學會公司專用 Log 格式或術語。

模型量化 (Quantization)： 犧牲微小精度換取體積大幅縮小，實現純 CPU 或低階 GPU 跑大模型。

邊緣小語言模型 (SLMs)： 將 1B~8B 的模型部署在地端，專解敏感資料遮蔽等單一任務，確保資料絕不出境。

程式化提示詞 (DSPy 等)： 用 Python 宣告結構，讓框架幫你算出最佳 Prompt，取代容易失效的手寫提示詞。
