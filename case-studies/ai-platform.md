---
layout: page
title: "Case Study: AI Platform — 統籌 20 人 AI Lab 的研究落地之路"
permalink: /case-studies/ai-platform/
---

## 背景

iKala AI Lab 是一支 20 人的 AI 團隊，負責 KOL Radar 平台上所有 ML 模型的研發與落地。團隊維護超過 30 個模型，涵蓋搜尋引擎、推薦系統、文本分類、電腦視覺、NLP 基礎設施與 LLM 標註平台，服務橫跨 Instagram、YouTube、Facebook、TikTok、Threads、Twitter 六大社群平台，覆蓋台灣、日本、韓國、印尼、越南、泰國、馬來西亞共七個市場。

技術棧分為三層：應用層面對用戶（搜尋、推薦、標籤分類、業配偵測、輿情分析）、AI 服務層包含 Hermes 搜尋引擎、GeneralTag 主題分類、TextAction 行為萃取等 21 個微服務、基礎設施層則是 Elasticsearch、BigQuery/Trino/Iceberg、5 台 GPU 機器（A6000/RTX 6000 Ada/RTX 3090）加上 OpenAI/Gemini/Claude API。

身為 AI Lab 的 TPM，我的角色是把研究成果轉化為可量化的產品指標提升——不只是讓模型跑起來，而是讓模型真正改善用戶體驗。

<div class="mermaid">
graph TD
    subgraph 應用層
        SR["搜尋"]
        REC["推薦"]
        TAG["標籤分類"]
        SP["業配偵測"]
    end

    subgraph AI 服務層
        HER["Hermes<br/>搜尋引擎"]
        GT["GeneralTag<br/>主題分類"]
        TA["TextAction<br/>行為萃取"]
        SG["Similar KOL<br/>推薦系統"]
    end

    subgraph 基礎設施層
        ES[("Elasticsearch")]
        TRINO[("Trino/Iceberg")]
        GPU["GPU x5 台"]
        LLM["LLM API"]
    end

    SR --> HER
    REC --> SG
    TAG --> GT
    SP --> TA
    HER --> ES
    SG --> TRINO
    GT --> LLM
    TA --> GPU

    style SR fill:#4a90d9,color:#fff
    style REC fill:#4a90d9,color:#fff
    style TAG fill:#4a90d9,color:#fff
    style SP fill:#4a90d9,color:#fff
    style HER fill:#e8a838,color:#fff
    style GT fill:#e8a838,color:#fff
    style TA fill:#e8a838,color:#fff
    style SG fill:#e8a838,color:#fff
    style ES fill:#e05d6f,color:#fff
    style TRINO fill:#e05d6f,color:#fff
    style GPU fill:#5bbf72,color:#fff
    style LLM fill:#5bbf72,color:#fff
</div>

---

## 核心模型群

AI Lab 的核心產出可以歸納為四大模型系統，每一個都經歷了多次技術迭代：

### Hermes 搜尋引擎 — BM25 + MiniLM 64d 混合架構

Hermes 是 KOL Radar 的核心搜尋引擎，採用 BM25 + Embedding 的混合搜尋架構，支援七種語言的 Analyzer（中文 ik_max_word、日文 kuromoji、韓文 nori、泰文 thai 等）。

核心研究問題是 embedding 壓縮：將 paraphrase-multilingual-MiniLM-L12-v2 的 384 維向量壓到 64 維，儲存空間降至六分之一，推論延遲從 387ms 降到 138ms，而搜尋品質（NDCG@10/100）反而優於原始維度。降維實驗比較了 PCA、Query Generation 重訓、Triplet Training 三種方法，最終選擇 BM25 hard negatives 的 triplet training 路線。

查詢策略採用 multi_match 的 best_fields 模式（tie_breaker=0.5），讓跨語言搜尋自然發生——一個查詢同時命中中文和英文內容。

### Similar KOL 推薦 — DeepWalk + FAISS 三版演進

推薦系統從 V1 到 V3 經歷了完整的技術路徑演進：

- **V1（TF-IDF）**：純內容特徵，pairwise similarity 矩陣。YouTube 的 1M x 1M 矩陣吃 120GB 記憶體，用 KNN clustering 分 30 群解決。但根本問題是只看內容不看行為。
- **V2（DeepWalk）**：引入 GA4 用戶瀏覽行為序列，跑 DeepWalk 產生 32 維 embedding（walk_length=5, num_walks=20, window_size=3），FAISS L2 搜尋取 Top-100。A/B Test 各 30% 流量，V2 勝出。設計五階段 fallback 策略解決冷啟動。
- **V3（Trino/Iceberg）**：演算法不動，資料源從 BigQuery 搬到 Trino/Iceberg。SQL 的 LEFT JOIN 改 INNER JOIN，台灣 IG 資料量下降 98%，記憶體可控。

### GeneralTag 標籤分類 — BERT 到 LLM 的技術世代交替

對 180M+ 貼文做 Level 1 + Level 2 階層式多標籤分類（HMTC），三年換了三代技術：

- **Phase 1（HMCN）**：NeuralClassifier 框架，Precision 0.924、Recall 0.884、F-score 0.903。
- **Phase 2（多語言擴展）**：從純中文擴展到 6 種語言，語意邊界問題浮現——周杰倫穿 Nike 被標「運動」而非「時尚」，情侶互稱「寶寶」被標「親子」。
- **Phase 3（LLM 化）**：用 LLM fine-tuning 取代傳統模型，Dynamic Tags 從 DB 動態讀取、不再寫死在模型裡，支援 zero-shot 分類新類別。

### TextAction 行為萃取 — mt5 seq2seq

基於自研 asym-t5-small encoder-decoder 模型，萃取 KOL 貼文中的六大行為類型。模型規模控制在 128M 以下，在 benchmark 上跑贏 mt5-small-trim 和 google/t5-v1_1-small，推論速度最快（長文 512 seq 優勢明顯）。TextAction-En 使用 reward model 做二次品質過濾，accuracy 提升 25%。

---

## 技術選型決策

在 AI Lab 的兩年多裡，我參與了幾個關鍵的技術選型決策。這些決策背後的邏輯可以歸納為五種模式：

### MiniLM vs OpenAI Embedding

Hermes 搜尋引擎比較了自訓 MiniLM 64 維和 OpenAI text-embedding-3-small（64d / 256d）。結果：MiniLM 在社群媒體文本的 NDCG 上表現更好，推論速度 4.9ms 遠快於 mcontriever 的 9.7ms。原因是社群媒體文本太特殊——emoji 大量出現、hashtag 污染、中英日混雜——通用 embedding 處理不好這些邊緣情境，domain-specific fine-tune 的效果差距很大。

**決策原則：先通用後特化。** 用通用模型建立 baseline，再用 domain-specific 訓練超越。

### LLM Labeling vs 傳統 ML

GeneralTag 從 HMCN（F-score 0.903）轉向 LLM，不是因為傳統模型不夠好，而是架構層級的限制。HMCN 做的是特徵空間的線性分割，碰到語意邊界就力不從心——Nike 和「運動」在訓練資料裡高度共現，模型學會了這個捷徑，但無法理解「周杰倫穿 Nike 配牛仔褲拍時尚照」和「健身教練穿 Nike 跑步」的語境差異。

NLPFactory 的標準產出路徑：GPT-4 few-shot 標註（離線）→ 小模型微調（1-2 天）→ ONNX 打包部署（即時推論），整個 pipeline 約 8 天，比人工標註快 3.5 倍。

**決策原則：先離線後即時。** 用大模型離線產出高品質標註，再蒸餾成小模型做即時推論。

### BigQuery → Trino/Iceberg（$466 vs $1200/月）

Similar KOL V3 把資料源從 BigQuery 搬到 Trino/Iceberg，月度成本從 $1,200 降到 $466。但搬家的真正動機不只是省錢——Trino/Iceberg 的架構讓批次計算和 checkpoint 更靈活，GKE spot node 被回收時能接續執行，而不是從頭跑 17 小時。

<div class="mermaid">
graph LR
    subgraph "五種選型模式"
        P1["先簡後深<br/>fastText → DL → LLM"]:::p1
        P2["先通用後特化<br/>通用 Embedding → MiniLM 64d"]:::p2
        P3["先單語後多語<br/>zh → ja → en → SEA"]:::p3
        P4["先規則後 AI<br/>BM25 → BM25+Vector"]:::p4
        P5["先離線後即時<br/>GPT-4 標註 → ONNX 部署"]:::p5
    end
    style P1 fill:#5bbf72,color:#fff
    style P2 fill:#4a90d9,color:#fff
    style P3 fill:#e8a838,color:#fff
    style P4 fill:#9b59b6,color:#fff
    style P5 fill:#e05d6f,color:#fff
</div>

---

## 品質保證 — A/B Testing 與實驗文化

AI 模型上線不靠拍腦袋，靠的是 A/B Testing 框架和品質指標。

### Planet / Layer / Bucket 分流架構

我們設計了三層分流架構，靈感來自 Google 2010 年 KDD 論文的 Overlapping Experiment Infrastructure：

- **Bucket（桶）**：用戶均勻分到 Q 個 bucket，是分流最小單位
- **Layer（世界）**：N 個桶組成一層，不同層用獨立 hash 函數分桶（分層正交），統計上互相獨立
- **Planet（星球）**：所有層構成一顆星球，有生命週期管理

這個架構讓我們能同時跑多個互不干擾的實驗。Similar KOL V2 就是用 30% / 30% 的流量分配做 A/B Test，確認 DeepWalk 方案勝出。

### CSQI 搜尋品質指數

Hermes 的品質衡量使用 CSQI（搜尋品質指數）和 NDCG@10/100 做 offline evaluation。Search Quality 是 AI Lab 最長壽的專案（2023-06 到 2025-05），北極星指標定義為「增加每用戶的 influencer collections」。

### 實驗驅動的決策流程

每個實驗的決策流程：SRM 檢定 → 護欄指標檢查 → 主指標是否達 MDE 且統計顯著。護欄指標（crash rate、latency、revenue）是安全網，即使主指標正向，護欄不過也不能上線。

NLPFactory 的每個模型上線前都有 100 筆人工驗收的品質閘門，TextAction-En 有 reward model 二次過濾，General Tag 有 LLM-as-a-Judge 月度稽核。

---

## 數據飛輪 — AI 功能如何回饋改善模型

<div class="mermaid">
graph TD
    A["用戶行為<br/>搜尋 / 點擊 / 收藏"] --> B["行為數據收集<br/>GA4 / Amplitude"]
    B --> C["模型訓練<br/>DeepWalk / Hermes / ALS"]
    C --> D["推薦 / 搜尋品質提升"]
    D --> E["更好的用戶體驗"]
    E --> A

    B -.->|"冷啟動"| F["新 KOL 無行為歷史<br/>五階段 fallback"]
    C -.->|"跨系統依賴"| G["General Tag 錯誤<br/>沿依賴鏈傳遞"]
    D -.->|"Feedback 延遲"| H["批次計算<br/>非即時更新"]

    style A fill:#4a90d9,color:#fff
    style B fill:#e8a838,color:#fff
    style C fill:#9b59b6,color:#fff
    style D fill:#50c878,color:#fff
    style E fill:#2ecc71,color:#fff
    style F fill:#e74c3c,color:#fff
    style G fill:#e74c3c,color:#fff
    style H fill:#e74c3c,color:#fff
</div>

數據飛輪理論上很美：用戶行為改善模型，模型改善體驗，更多用戶帶來更多數據。實務上，飛輪的每個環節都有摩擦力。

**Similar KOL 的飛輪是最成功的案例。** V1 純靠內容特徵，V2 把 GA4 用戶瀏覽行為序列餵進 DeepWalk，學出「哪些 KOL 常被一起看」的信號。用戶的瀏覽行為直接改善推薦品質，推薦品質提升帶來更多瀏覽——飛輪轉起來了。

**Hermes 的飛輪還沒完全閉環。** 搜尋行為數據有（Amplitude 記錄了 Search for KOL、Visit KOL Detail 等事件），但要把點擊數據轉化成 relevance label 需要處理 position bias，這個 offline evaluation pipeline 的自動化程度還不夠。

**ALS 推薦是飛輪做得最完整的。** 基於 247,833 筆業配紀錄做協同過濾，但它的資料來源是 TextSponsor 模型的輸出——如果業配偵測誤判，錯誤數據會進入訓練集，沿著依賴鏈傳遞。

### PM 在飛輪裡的三件事

身為 TPM，我在飛輪中做的不只是寫 spec：

1. **定義有效行為信號**：點擊後停留超過 5 秒才算 positive signal，純曝光不進訓練集。這個閾值是看了用戶行為分佈後決定的。
2. **設計讓 feedback 跑回來的 UI**：收藏按鈕放在卡片上而不是詳情頁裡，「不感興趣」用手勢滑掉。這些小設計直接影響飛輪的數據品質。
3. **決定 retrain 觸發時機**：不是技術排程驅動，是業務信號驅動——搜尋點擊率連續兩週下降，才是真正需要 retrain 的時機。

---

## 成果

| 指標 | 改善幅度 | 具體方法 |
|------|----------|----------|
| 平台搜尋準確度 | **+30%** | MiniLM 64d 混合搜尋 + 七語系 Analyzer |
| 功能採用率 | **+20%** | Similar KOL V2 A/B Test 驗證 + 數據飛輪 |
| 模型維護成本 | **-20%** | ONNX 統一部署 + Trino/Iceberg 遷移（$466 vs $1200） |
| Embedding 儲存 | **降至 1/6** | 384 維壓縮到 64 維，延遲從 387ms 降到 138ms |
| 標籤分類 F-score | **0.903** | HMCN 到 LLM 的技術世代交替，語意邊界問題解決 |
| 標註效率 | **3.5 倍** | NLPFactory pipeline：GPT-4 標註 → 小模型微調 → ONNX |

---

## 反思

三個我在這個過程中學到的事：

**技術選型的「漸進式複雜化」原則。** 21 個服務沒有一個第一版用最複雜的技術。fastText 能解決的問題不需要 BERT，BM25 夠用就先上線。每次升級都有數據驅動：品質不夠才換模型、延遲太高才做壓縮、新市場才加新語言。

**推薦系統最被低估的工作不是調模型，是把資料搞乾淨。** SQL 改一行降 98%、DataFrame index 對齊的 bug 導致數量異常——這些才是日常。模型再好，跑在錯的資料上也沒意義。

**數據飛輪不是建好就不管的東西。** 行為數據品質會退化，模型效果會漂移，用戶使用模式會改變。飛輪的每個環節都需要人盯，而「什麼時候該介入」的判斷，比任何 hyperparameter 都難。

---

## 相關文章

- [相似網紅怎麼算？從 TF-IDF 到 DeepWalk 的三次演進](/2026/02/28/01-similar-kol-三次演進/)
- [搜尋底層技術：BM25 + Embedding 混合架構與 7 語系 Analyzer](/2026/03/02/03-搜尋底層技術-hermes/)
- [標籤系統從 HMTC 到 LLM：一次技術世代交替](/2026/03/03/04-標籤系統-generaltag/)
- [AI Lab 技術全景：21 個服務的三層架構選型](/2026/03/30/32-ailab-技術全景/)
- [AI 產品的數據飛輪：怎麼讓用戶行為餵回模型](/2026/04/01/34-數據飛輪/)
- [AB 測試怎麼做？分流、分桶、平行世界的設計邏輯](/2026/03/12/13-ab-test/)
