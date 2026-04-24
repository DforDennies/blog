---
layout: post
title: "標籤系統從 HMTC 到 LLM：一次技術世代交替"
date: 2026-01-07
categories: [AI/ML 技術]
tags: [Machine Learning, NLP, AI, Deep Learning]
author: Dennies Hong
---

周杰倫的貼文被標成「運動」，因為他代言了運動品牌。

模型看到 Nike、adidas 就觸發「運動」標籤，但那篇貼文其實是時尚穿搭。情侶之間叫對方「寶寶」，模型標成「親子」。接吻的文章被標成「呼吸道疾病」。這些案例聽起來好笑，但每一個都代表標籤系統在語意邊界上的失敗——而這些標籤直接影響搜尋結果和推薦品質。

GeneralTag 是 KOL Radar 的核心標籤系統，對每篇 KOL 貼文做 Level 1 + Level 2 的階層式多標籤分類（HMTC）。從 2023 年的傳統模型到 2026 年的 LLM 方案，走了三年，換了三代技術。

## 分類體系：三層結構

<div class="mermaid">
graph TD
    ROOT["Root"]:::root --> L1A["健康"]:::l1
    ROOT --> L1B["遊戲"]:::l1
    ROOT --> L1C["美妝時尚"]:::l1
    ROOT --> L1D["...10+ 大類"]:::l1
    L1A --> L2A["美髮美妝"]:::l2
    L1A --> L2B["運動減肥"]:::l2
    L1A --> L2C["飲食保健"]:::l2
    L1A --> L2D["...22 子類"]:::l2
    style ROOT fill:#2c3e50,stroke:#1a252f,color:#fff
    style L1A fill:#e74c3c,stroke:#c0392b,color:#fff
    style L1B fill:#3498db,stroke:#2980b9,color:#fff
    style L1C fill:#9b59b6,stroke:#8e44ad,color:#fff
    style L1D fill:#95a5a6,stroke:#7f8c8d,color:#fff
    style L2A fill:#f39c12,stroke:#e67e22,color:#fff
    style L2B fill:#1abc9c,stroke:#16a085,color:#fff
    style L2C fill:#e67e22,stroke:#d35400,color:#fff
    style L2D fill:#bdc3c7,stroke:#95a5a6
</div>

先看問題的規模。


GeneralTag 的分類體系是 Root → Level 1 → Level 2 的三層階層結構，10 個以上的 Level 1 大類。以健康類為例，底下就有 22 個子類別：美髮美妝、皮膚健康、身心健康、口腔保健、運動減肥、糖尿病、飲食保健、婦科、癌症 — 光列出來就知道這些類別之間的邊界有多模糊。

一篇貼文可以同時屬於多個類別，這是 multi-label 的特性。一篇講地中海飲食的文章可以同時是「飲食保健」和「婦科」（因為提到了懷孕和哺乳期的飲食建議），這種多標籤的情況是正確的。但模型要能區分「真的多標籤」和「誤判」。

訓練資料的規模：遊戲類 62,000 筆、健康類 80,107 筆、美妝時尚類 39,229 筆。每個類別內部的分布極度不均 — 遊戲類裡「其他」有 42,054 筆，但「玩具收藏」只有 753 筆。這種 class imbalance 是分類任務的常見痛點。

## Phase 1：HMCN 起手

2023 年 4 月，第一版 GeneralTag 從純中文開始。


模型選型做了一輪實驗。用的是 Tencent 開源的 NeuralNLP-NeuralClassifier 框架，裡面支援 TextRNN、TextRCNN、HMCN 等多種架構。最終 HMCN 勝出：Precision 0.924、Recall 0.884、F-score 0.903。

HMCN（Hierarchical Multi-label Classification Network）的特點是 loss function 裡有 recursive regularization，會懲罰子類別預測和父類別預測不一致的情況。比如模型如果預測了「飲食保健」（Level 2）但沒預測「健康」（Level 1），regularization 會拉高「健康」的機率。

我們也試過 OOD（Out-of-Distribution）偵測，用二郎神（MegatronBert 1.3B）和 mDeBERTa-v3 做 entailment model。想法是：如果一篇貼文跟所有類別的 entailment score 都低於 threshold 0.7，就判定它不屬於任何類別。但最終沒有採用，原因是推論時間太長，而且實驗顯示不提升效能。

Entailment 的嘗試還是留下了有用的結論。用「和遊戲有關」當 hypothesis 測試一篇電競新聞，ENTAILMENT 分數 0.7425，合理。同一篇新聞用「和金融有關」測試，分數 0.5652 — 因為標題裡有「金控」兩個字。模型能抓到這種弱關聯，但 threshold 設在哪裡就成了問題。最終決定用 Level 1 + Level 2 entailment 都低於 0.5 才排除。

## Phase 2：語意邊界的挑戰

v0.1.0 到 v1.0 的演進主要是多語言擴展和更多類別覆蓋。從純中文擴展到 en、ja、vi、th、ms 六種語言，走的是 Autotag 的 pipeline：Understanding → Setup → Model tuning → Stage 2/3 validation。

但真正的問題在 Phase 2 浮現：語意邊界。


這批誤判案例很有代表性：

- 周杰倫貼文被標「運動」— 他代言了運動品牌，但貼文主題是穿搭
- 情侶互稱「寶寶」被標「親子」— 暱稱不等於真有小孩
- 接吻相關文章被標「呼吸道疾病」— 因為涉及口腔

這些案例的共同特徵是：模型學到的是關鍵字的統計關聯，而不是真正的語意理解。Nike 和「運動」在訓練資料裡高度共現，模型就學會了這個捷徑。但周杰倫穿 Nike 的照片是時尚內容，不是運動內容。

錯誤分析也揪出一些「合理誤判」— 嚴格來說模型沒錯，是標註體系的邊界不夠清楚。Xbox 紀錄片被標成「電視影劇」而非「電玩遊戲」，紅肉文章被標成「心血管疾病」而非「飲食保健」— 這些其實都有道理，只是 ground truth 只標了一個類別。

Phase 2 的工作重點放在 DataCenter 儲存優化和 StarRocks 對接，EDA 和標註平台也在這個階段建起來。


## Phase 3：LLM 接管

2026 年，GeneralTag 走向 LLM 化。

方向很明確：用 LLM fine-tuning 取代傳統 HMCN 模型。具體計劃包括三個面向：

**LLM Fine-tuning**：先用 small VLM 做初步訓練，把開源模型打包上傳 Google Artifact Registry，不同 LLM 模型拆到不同 docker image 裡。這樣做的好處是可以獨立更新和擴展不同模型，不用每次改一個模型就重新部署整個 pipeline。

**Dynamic Tags**：Level 1 和 Level 2 的 tags 從 DB 動態讀取，不再寫死在模型裡。這解決了一個長期痛點 — 每次新增或調整類別都要重新訓練模型。動態標籤讓分類體系可以隨業務需求靈活調整，同時要防止過度多樣化。

**資料準備**：先從 Instagram 標註開始，由 Hank 對接線上資料。Patty 做 EDA（探索性資料分析），之後接著標 Threads 平台的資料。

為什麼 LLM 能解決語意邊界的問題？因為 LLM 有更深的語境理解能力。傳統模型看到「Nike」就觸發「運動」，但 LLM 能理解整篇貼文的脈絡 — 周杰倫穿 Nike 配牛仔褲拍時尚照，跟健身教練穿 Nike 跑步機上揮汗，是完全不同的語境。LLM 的 attention 機制讓它能把品牌名和周圍的穿搭、時尚等語境連結起來，而不只是做關鍵字 pattern matching。

## 從 HMCN 到 LLM 的技術轉折

<div class="mermaid">
timeline
    title GeneralTag 技術演進
    2023 : Phase 1 HMCN
         : F-score 0.903
         : 純中文起步
    2024-2025 : Phase 2 多語言擴展
              : 6 種語言
              : 語意邊界問題浮現
    2026 : Phase 3 LLM 化
         : Dynamic Tags
         : Zero-shot 分類
</div>

回頭看這三年的演進：

2023 年用 NeuralClassifier 的 HMCN，F-score 0.903，在大部分場景夠用。模型輕量，推論快，部署簡單。但它本質上是在做特徵空間的線性分割，碰到語意邊界就力不從心。

2024-2025 年的多語言擴展把覆蓋面推到六種語言，但每增加一種語言都需要重新準備標註資料、調整 tokenizer、做 stage validation。DiffCSE 自研編碼器（基於 ikala pretrained Electra）的訓練時間大約 5-7 小時，還算可控。

2026 年轉向 LLM，是因為前兩代的架構已經觸及天花板。語意邊界問題用更大的傳統模型解決不了 — 不是模型不夠大，是架構不對。HMCN 再怎麼加 regularization，也無法像 LLM 那樣理解「周杰倫穿 Nike 是時尚還是運動」這種需要常識推理的判斷。

我認為這就是典型的「技術世代交替」——不是漸進式改良，是架構層級的跳躍。HMCN 在它的設計邊界內已經做到 F-score 0.903，但語意邊界的問題不是加更多 regularization 能解的。

---

Phase 3 還在進行中，Dynamic Tags 的設計是我最想看到結果的部分。從 DB 動態讀取標籤意味著 LLM 在 inference 時會遇到從未見過的新類別——本質上是 zero-shot 分類。LLM 的 in-context learning 理論上可以處理，但生產環境裡，新增一個類別後的分類品質到底能到什麼水平？會不會影響既有類別的準確度？這些還需要更多實驗數據，目前沒有答案。

---

*Dennies Hong — AI/ML Product Manager / 寫技術、寫產品、寫真實經歷。*
