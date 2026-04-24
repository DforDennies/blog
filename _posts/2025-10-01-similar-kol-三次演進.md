---
layout: post
title: "相似網紅怎麼算？從 TF-IDF 到 DeepWalk 的三次演進"
date: 2025-10-01
categories: [AI/ML 技術]
tags: [Machine Learning, NLP, AI, Deep Learning]
author: Dennies Hong
---

去年有一次，業務跑來跟我說：「客戶反映相似網紅推薦不準，推出來的人跟原本那位完全不像。」我第一反應是去查 General Tag 準確度，結果發現問題不在標籤——我們算「相似」的方式，從一開始就換了三次完全不同的技術路線。

這篇聊的是我在 iKala AI Lab 參與 Similar KOL 推薦系統（內部代號 CHAOS）從 V1 到 V3 的過程。比較像 TPM 視角的回顧：每一版為什麼要改、改了什麼、踩了哪些坑。

## 先定義問題：「相似」是什麼

<div class="mermaid">
graph LR
    V1["V1<br/>TF-IDF 內容相似"]:::v1 -->|"A/B Test 驗證"| V2["V2<br/>DeepWalk 行為相似"]:::v2
    V2 -->|"資料源搬家"| V3["V3<br/>Trino/Iceberg<br/>效能優化"]:::v3
    style V1 fill:#ff6b6b,stroke:#c0392b,color:#fff
    style V2 fill:#4ecdc4,stroke:#1abc9c,color:#fff
    style V3 fill:#45b7d1,stroke:#2980b9,color:#fff
</div>

先說結論：這題沒有標準答案，不同的定義直接決定技術選型。

KOL Radar 是幫品牌找網紅合作的平台。品牌在看某位 KOL 的詳情頁時，系統推薦最多 100 位「相似的網紅」，幫他們擴展合作名單。但「相似」到底是什麼？內容主題像？受眾重疊？粉絲量級接近？

我們追蹤的核心指標是「有點比」——推薦的 KOL 中有多少比例真的被點開。這個數字很殘酷，用戶不會跟你解釋為什麼不點，你只能從 CTR 反推推薦品質。


## V1：能用，但方向就是不太對

先講結果：V1 靠貼文內容算相似度，上線後能用，但推薦品質被 V2 全面超越。

做法是標準 NLP 路線。每位 KOL 的貼文跑 TF-IDF 產生特徵向量，加上 hashtag 共現、mention 共線，再混入粉絲數和互動率。四組特徵加權合併，pairwise distance 算出整張相似度矩陣。


規模一上去就出事了。YouTube 的 content matrix density 高達 91%，similarity matrix 1M × 1M 要吃大約 120GB 記憶體。最後用 KNN clustering 解——先分 30 個 cluster，只在同 cluster 內計算。IG 台灣區大概 6.1 萬 KOL，分完勉強跑得動。

V1 根本性的問題：它只看「內容」，不看「用戶行為」。兩個 KOL 貼文主題可能很像，但品牌方實際上根本不會一起考慮他們。反過來，兩個看似不相關的 KOL，因為受眾重疊，品牌方經常一起瀏覽。這件事 V1 完全捕捉不到。

## V2：A/B Test 確認贏了，因為換了問問題的方式

V2 的結論先講：上線後 A/B Test 流量各 30%，V2 贏了。原因我認為很直覺——V1 問的是「這兩個人的貼文像不像」，V2 問的是「用戶有沒有一起看這兩個人」。後者更貼近實際需求，因為品牌選 KOL 的邏輯不是比對文字，是看自己和同行都在關注誰。


核心改動是引入 DeepWalk。從 GA4 事件中拿到用戶瀏覽 KOL 的行為序列，建成一張圖，跑 DeepWalk 產生 32 維的 embedding。如果用戶在同一個 session 裡先看了 KOL A 再看 KOL B，A 和 B 之間就存在某種關聯。

參數：walk_length=5、num_walks=20、window_size=3。靈感來自 Airbnb 2018 年那篇 "Real-time Personalization using Embeddings for Search Ranking"，根據我們的資料規模調過。

<div class="mermaid">
graph TD
    GA4["GA4 瀏覽事件"]:::data --> SEQ["用戶行為序列"]:::process
    SEQ --> GRAPH["KOL 關聯圖"]:::process
    GRAPH --> DW["DeepWalk<br/>32維 Embedding"]:::model
    DW --> FAISS["FAISS L2 搜尋"]:::output
    FAISS --> TOP100["Top-100 推薦"]:::output
    style GA4 fill:#f39c12,stroke:#e67e22,color:#fff
    style SEQ fill:#e74c3c,stroke:#c0392b,color:#fff
    style GRAPH fill:#9b59b6,stroke:#8e44ad,color:#fff
    style DW fill:#3498db,stroke:#2980b9,color:#fff
    style FAISS fill:#2ecc71,stroke:#27ae60,color:#fff
    style TOP100 fill:#1abc9c,stroke:#16a085,color:#fff
</div>

光靠 embedding 不夠，新 KOL 沒行為資料（冷啟動）。V2 設計了五階段混合策略依序填滿 Top-100：


1. **Stage 1**（Content + CF）：相同 General Tag 的候選，FAISS L2 距離搜索
2. **Stage 2**（Tag Random）：遍歷所有標籤，同標籤候選隨機採樣
3. **Stage 3**（Pure CF）：純 embedding 相似度，不管標籤
4. **Stage 4**（Random Fallback）：前面填不滿，隨機補
5. **Stage 5**：還不夠就停

Stage 1 保證品質，後面解決覆蓋率。最終分數用 Tag 重疊的 Jaccard Similarity。

## V3：演算法沒動，但 SQL 改一行降了 98%

V3 幾乎沒改演算法，但整個資料源搬家了——BigQuery 切到 Trino/Iceberg，ID 從 KOL UUID 切到 DC encode_id。

搬家過程暴露了不少問題。最頭痛的是 IG-JP 組合：日本市場大概 59 萬 KOL，跑一次要 17 小時。GKE spot node 跑 17 小時，中間被回收的機率很高。解法是把 find_top（佔 75% 計算時間）拆 batch，加 checkpoint 到 GCS，中斷了能接續。


另一個我認為最值得講的優化：V3 把 SQL 的 LEFT JOIN 改成 INNER JOIN，光這一步台灣 IG 資料量下降 98%，記憶體從爆炸降到 10GB 可控。認真說，這種改動在實務中比調 hyperparameter 常見太多了，但很少有人會拿出來講。

V3 初期只跑台灣，其他國家維持 V2。2025 年 10 月的決策——資源有限，先確保主要市場穩定，演算法優化另外開票。

## 跑完三版的體感

三版的演進其實是推薦系統的典型路徑：content-based 快速上線 → collaborative filtering 提升品質 → 基礎設施技術債清理。跑過才知道，教科書上的路徑圖省略了很多中間的坑。

推薦「準不準」很難量化。有點比、CTR 能看趨勢，但用戶跟你說「不準」的時候，背後原因可能是標籤品質、冷啟動、或者他對「相似」的定義根本跟你不一樣。這題到現在也沒有完全解。

Spot Node 省錢但要設計好容錯。17 小時跑在 spot 上，被回收只是時間問題。這件事不是演算法團隊要想的，但如果沒有人把排程和容錯串起來，模型再好也跑不完。

我認為推薦系統最被低估的工作不是調模型，是把資料搞乾淨。SQL 改一行降 98%、DataFrame index 對齊的 bug 導致 V2 數量異常——這些才是日常。

三版做下來，「相似」的定義從內容相似、行為相似，到現在用戶又開始反映不夠準。下一步可能要把更多網紅屬性拉進來，或者乾脆讓用戶自己定義什麼叫相似。

但真正讓我反覆想的是：推薦系統的「準」到底該由誰定義？工程師看 CTR，業務看客戶反饋，PM 看留存率——三個人看到的「準」可能完全不同。這個對齊問題，比調任何 hyperparameter 都難。

---

*Dennies Hong — AI/ML Product Manager / 寫技術、寫產品、寫真實經歷。*
