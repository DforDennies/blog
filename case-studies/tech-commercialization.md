---
layout: page
title: "Case Study: 技術商業化 — 把 AI 翻譯成客戶聽得懂的語言"
permalink: /case-studies/tech-commercialization/
---

> Engineers speak model accuracy, clients speak ROI. The translation gap kills deals.

在 iKala AI Lab 的兩年，我最常做的事不是寫 PRD，而是**翻譯**。把 ML 工程師口中的 F1 score 翻譯成客戶聽得懂的商業價值，把實驗時間軸翻譯成 Sprint 可追蹤的 milestone，把技術風險翻譯成商業決策者能評估的代價。

這篇 Case Study 記錄我怎麼在技術與商業之間搭建翻譯層，以及三個具體案例的商業化過程。

---

## 問題：兩個世界的語言不通

AI Lab 的模型很強，但「強」這個字對不同人意味著完全不同的東西。

ML 工程師說「F1 score 達到 95%」，客戶聽到的是：「所以這東西到底能幫我省多少錢？」PM 說「預計下個 Sprint 上線」，工程師聽到的是：「你在逼我承諾一個不確定的結果。」業務說「客戶想要一個 AI 解決方案」，工程師聽到的是：「又來了一個不知道自己要什麼的需求。」

這不是溝通能力的問題，是**認知框架的根本差異**。PM 用線性時間思考（Sprint 1 → Sprint 2 → Sprint 3），ML 工程師用條件分支思考（實驗 A 結果好 → 繼續，結果差 → 換方向）。兩邊都沒有錯，但如果沒有人做翻譯，技術價值就永遠停留在實驗室裡。

<div class="mermaid">
graph TB
    subgraph "ML 工程師的世界"
        M1["F1 score 95%"]
        M2["推論延遲 138ms"]
        M3["ONNX 模型 18M 參數"]
        M4["accuracy 提升 25%"]
    end
    subgraph "PM 翻譯層"
        T1["每 100 筆結果<br/>只有 5 筆判斷錯誤"]
        T2["使用者搜尋後<br/>0.14 秒內看到結果"]
        T3["部署在 CPU 上<br/>不需要 GPU 成本"]
        T4["搜尋結果誤判率<br/>大幅降低"]
    end
    subgraph "客戶 / 業務的世界"
        B1["品牌安全有保障"]
        B2["即時搜尋體驗"]
        B3["每文件成本 $0.03"]
        B4["找錯 KOL 的<br/>機率更低"]
    end
    M1 --> T1 --> B1
    M2 --> T2 --> B2
    M3 --> T3 --> B3
    M4 --> T4 --> B4
    style M1 fill:#e67e22,color:#fff
    style M2 fill:#e67e22,color:#fff
    style M3 fill:#e67e22,color:#fff
    style M4 fill:#e67e22,color:#fff
    style T1 fill:#3498db,color:#fff
    style T2 fill:#3498db,color:#fff
    style T3 fill:#3498db,color:#fff
    style T4 fill:#3498db,color:#fff
    style B1 fill:#2ecc71,color:#fff
    style B2 fill:#2ecc71,color:#fff
    style B3 fill:#2ecc71,color:#fff
    style B4 fill:#2ecc71,color:#fff
</div>

---

## PM-ML 翻譯層：我怎麼搭這座橋

兩年、120+ 次 Weekly Sync-up，我整理出三個翻譯維度。

### 維度一：時間感知翻譯

ML 工程師說「兩週」，跟 PM 說「兩週」，是完全不同的意思。

PM 的兩週是確定性的：有 ticket、有驗收條件、有上線日期。ML 的兩週是條件性的：「如果實驗結果好，兩週；如果不好，方向要換。」

我的做法是把 ML 專案拆成兩種 ticket：**研究 milestone**（「完成 reward model 二次過濾，提交評估報告」）和**交付 ticket**（「模型上線、接入產品」）。前者的完成標準是「有結果、有報告」，不是「模型達標」；後者才排進 Sprint，而且只在研究 milestone 完成後才排。

以 TextAction-En 為例，從 2024 年 10 月啟動到 2025 年 3 月完成，中間有個關鍵發現——加入「空標」後 accuracy 提升 25%，這不是一開始能預測的。如果我在 10 月就問「幾月交付」，答案一定是錯的。但如果我在每個實驗節點問「這個結果告訴我們什麼、下一步往哪走」，就能把研究進度翻譯成比較真實的 timeline。

### 維度二：指標翻譯

ML 工程師報 accuracy、F1、NDCG。這些數字很重要，但 PM 直接去討論意義不大——你不知道 0.85 是好還是不好、差距 0.03 值不值得再跑一週。

我學會的問法是：「這個版本跟上一個版本比，使用者的體感會不會有差別？」以及「這個準確率，在 production 裡大概每 100 個結果會有幾個是錯的？」

Hermes 搜尋引擎的例子：v1.5 把 embedding 維度從 768 壓到 64，推理時間從 387ms 降到 138ms。技術報告裡寫「推論速度提升 64%」。翻譯成產品語言：「搜尋結果現在快了將近三倍，使用者輸入完馬上看到結果。」後者才是業務能帶去見客戶的說法。

### 維度三：風險溝通

有些技術決策一旦做了就很難回頭。PM 如果不問，很容易在不對的時間點介入。

General Tag LLM 化是典型案例。2025 年 11 月，團隊做了一個重大轉向：不再迭代傳統 predict model，直接用 LLM 做線上標註。原因是 production 模型的品質遠低於 LLM，每次新資料進來，舊模型的低品質預測會覆蓋高品質標籤——「品質不對稱問題」，越晚解決壞資料越多。

但這需要 GPU 硬體升級，需要 PM 介入協調資源。如果我不懂品質不對稱問題是什麼、不懂時間窗口為什麼緊迫，就不知道要主動問「需要我協助什麼資源」。

---

## 案例 1：TextSponsor 業配偵測 — 技術功能變成商業武器

### 技術面

TextSponsor 在技術上是一個二元分類器：給定一篇社群貼文，判斷是否為業配。從繁體中文起步，花兩年擴展到六語言（zh/ja/en/vi/th/ms）、五平台（IG/FB/YT/Twitter/TikTok）。每個語言需要獨立的 EDA 和標註數據，因為不同文化的業配表達方式完全不同——台灣 KOL 寫「感謝 XX 品牌邀約體驗」，日本 KOL 寫「PR」，泰國 KOL 又是另一套。

### 商業翻譯

但客戶不在乎什麼是二元分類器。客戶在乎的是：

- **品牌安全**：「我想合作的 KOL 是不是已經在幫競品做業配？」
- **Campaign 驗證**：「我付了錢，KOL 到底發了沒有？發的內容算不算業配等級？」
- **定價參考**：「這個 KOL 過去接業配的頻率多高？報價合不合理？」

TextSponsor 不是一個獨立功能，它是整個數據管線的關鍵節點：TextSponsor 判斷「是業配」→ SPDA 歸因到品牌 → TextKnowledge 萃取品牌與商品名稱 → 流入 KOL 詳情頁的「業配歷史」和 ALS 推薦系統。

<div class="mermaid">
graph LR
    subgraph "技術管線"
        A["TextLangID<br/>語言偵測"] --> B["TextSponsor<br/>業配分類"]
        B --> C["SPDA<br/>品牌歸因"]
        C --> D["TextKnowledge<br/>品牌商品萃取"]
    end
    subgraph "商業價值"
        D --> E["業配歷史<br/>品牌安全查核"]
        D --> F["ALS 推薦<br/>精準媒合"]
        D --> G["K-Score<br/>影響力定價"]
    end
    style A fill:#94a3b8,color:#fff
    style B fill:#6366f1,color:#fff
    style C fill:#8b5cf6,color:#fff
    style D fill:#3b82f6,color:#fff
    style E fill:#10b981,color:#fff
    style F fill:#10b981,color:#fff
    style G fill:#10b981,color:#fff
</div>

技術語言：「六語言 NLP 二元分類器，管線式推論架構。」
商業語言：「品牌客戶可以一鍵查看任何 KOL 過去跟哪些品牌合作過、合作頻率、業配佔比——幫助判斷是否適合自己的品牌。」

2022 年 11 月 K-Score + Sponsored Post 偵測 BETA 上線，2023 年 2 月正式公開發布，成為 Premium / Enterprise 方案的付費功能。

---

## 案例 2：LLM 產品化 — 六個模組的嵌入策略

AI Lab 走的不是「接 API → 寫 prompt → 部署」的路，而是一條更長但成本低兩個數量級的路：**大模型生成標註資料，訓練小模型，打包成 ONNX，線上做即時推論**。

### NLPFactory 模式

整個流程大約 8 天完成一個任務（標註 3-5 天、微調 1-2 天、打包 1 天），每個任務花 $500-2000 USD，比人工標註快 3.5 倍。大模型和小模型的成本差距：速度差 250 倍，金額差 2.5 倍。

用這套模式產品化的模組：

| 模組 | 技術底層 | 商業價值 |
|------|---------|---------|
| TextAction | LLM 標註 → BERT 微調 | KOL 行為意圖分類，幫品牌找對的合作時機 |
| TextKeyphrase | LLM 標註 → 小模型 | 自動萃取貼文關鍵字，強化搜尋精準度 |
| TextKnowledge | LLM 標註 → NER | 品牌商品知識萃取，建構商業情報 |
| General Tag | fastText → HMTC → LLM | 171M 篇貼文自動分類，支撐整個搜尋與推薦 |
| Query Expansion | GPT-4.1-mini → Qwen3-4B LoRA | 搜尋查詢擴展（lex/vec/hyde），提升搜尋召回率 |
| Hermes 搜尋引擎 | 768d → 64d embedding 壓縮 | 搜尋延遲從 387ms 降到 138ms |

### VLM 蒸餾：成本砍半，性能 94-171%

更極端的案例是 VLM 蒸餾。120B 大模型在台灣任務上只有 10% 準確率（因為把「捷運」說成「地鐵」），但 8B 模型拿到 79.76%。我們用 8B 當 Teacher 蒸餾出 4B 的 Student，部署成本砍半（VRAM 24GB → 12GB），七項 benchmark 達成率 94-171%。

翻譯成商業語言：「同樣的 AI 能力，運算成本減半，而且真的聽得懂台灣人在說什麼。」

---

## 案例 3：企業交付 — 把平台能力翻譯成客戶方案

AI Lab 過去的產出都是內部服務。Client B 和 Client C 是第一次直接面對外部客戶。

### Client B（製造業）：工業文件解析 API

客戶需要把工業技術手冊（PDF）自動解析成結構化資料。我們用 MinerU 做 PDF 解析（Phase 1），再引入 GPT-4o 處理複雜表格和圖片（Phase 2）。

- F1 score：95%（結構化欄位）/ 71%（非結構化段落）
- 單文件成本：**$0.03 美元**——這個數字成為後續所有客戶報價的基準線
- 專案週期：12 個月，成功結案

商業翻譯：「你的工程師不用再手動讀 PDF 了。每份文件 1 塊台幣，準確率 95%。」

### Client C（健康管理）：LINE 聊天機器人

客戶需要一個 LINE 上的健康管理聊天機器人，底層直接用 40925 Agent 架構。Acceptance score 72.4（門檻 70），通過驗收。

但客戶決定不續約。不是技術問題——驗收通過了、功能交付了、SLA 也滿足了。

**教訓**：「通過驗收」不等於「客戶滿意」。72.4 通過了 70 的門檻，但通過門檻和讓客戶願意續約是兩件事。如果重來，我會在專案中期就開始做 business review——追蹤客戶內部使用率、stakeholder 滿意度、續約意向，而不是等驗收完才發現。

### 共用基礎設施的商業意義

兩個完全不同的案子（工業文件 vs 健康問答），底層都用了 NLPFactory + 40925 Agent 架構。翻譯成商業語言：「不是每個客戶都從零開始，我們有可復用的 AI 基礎設施，客製化的部分只需要調整 prompt 和資料源。」

---

## 成果

| 指標 | 數字 |
|------|------|
| 企業客製專案交付 | 2 個（Client B 成功結案、Client C 通過驗收） |
| 銷售 pipeline 中的原型 | 2 個 |
| 客戶專案成功率 | 70%+ |
| NLP 模組產品化 | 6 個模組嵌入 LLM 能力 |
| TextSponsor 語言覆蓋 | 6 語言、5 平台 |
| 單文件處理成本基準 | $0.03 USD |
| VLM 蒸餾部署成本節省 | 50%（VRAM 24GB → 12GB） |

技術商業化不是一次性的翻譯，是持續的溝通機制。每週 sync-up 前讀上週筆記、不懂的詞先記後查、主動把技術決策翻譯成商業影響、每月問一次「最大的技術限制是什麼」——這些不起眼的習慣，才是讓技術價值真正流向客戶的管道。

---

## 相關文章

- [PM 和 ML 工程師的翻譯層：我在 AI Lab 學到的溝通方式](/2026/04/05/38-pm-ml-翻譯層/)
- [業配偵測系統：六語言 NLP 的產品化之路](/2026/03/13/14-業配偵測-textsponsor/)
- [Prompt Engineering 到 LLM 產品化：從微調到上線的實戰流程](/2026/03/31/33-llm產品化/)
- [AI Lab 首次企業交付：從 API 到 LINE 聊天機器人](/2026/03/11/12-企業交付/)
- [VLM 蒸餾實戰：Qwen3-VL-4B 台灣版訓練全紀錄](/2026/03/01/02-vlm-蒸餾實戰/)
