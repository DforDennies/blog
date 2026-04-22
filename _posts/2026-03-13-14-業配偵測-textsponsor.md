---
layout: post
title: "業配偵測系統：六語言 NLP 的產品化之路"
date: 2026-03-13
categories: [AI/ML 技術]
tags: [Machine Learning, NLP, AI, Deep Learning]
author: Dennies Hong
---

KOL 一篇貼文值多少錢？先搞懂哪些是業配。

這是 TextSponsor 存在的理由。品牌客戶需要知道一個 KOL 過去接了多少業配、跟哪些品牌合作過、合作頻率多高。但社群平台不會幫你標記「這篇是業配」——你得自己從貼文內容裡判斷。

TextSponsor 是 iKala AI Lab 最早期的模型之一。從繁體中文起步，花了兩年擴展到六語言、五平台。這篇記錄這個系統的產品化歷程。

---

## 一個二元分類問題

TextSponsor 在技術上是一個二元分類器：給定一篇社群貼文，判斷它**是否為業配**。

聽起來簡單，但實際上有幾個挑戰：

1. **業配的定義模糊**。有些 KOL 會明確標記 #ad 或 #sponsored，但很多不會。有些是「軟業配」——看起來像日常分享，但其實是品牌合作。分類器需要處理這種灰色地帶
2. **語言差異大**。中文的業配用語（「合作」「邀約」「體驗」）和英文（#ad #sponsored #gifted）完全不同，日文又是另一套。不可能用一個模型打天下
3. **平台差異**。Instagram 貼文有 caption + hashtag，YouTube 有 title + description，TikTok 有短文字 + hashtag。每個平台的文本結構不同

---

## 演進時間線


### v1：繁體中文（pre-2022）

最早的版本只做繁體中文，跑在 Instagram 和 Facebook 上。AI Lab 的起步之作。

### 日文擴展（2023 Q1）

日本市場是 KOL Radar 的第二大市場，日文業配偵測是優先級最高的語言擴展。日文有自己的 EDA（Exploratory Data Analysis），因為日文的業配表達方式跟中文差很多——日文 KOL 更常用「PR」「タイアップ」（tie-up）等標記。

### 多語言爆發（2023 Q2-Q3）

2023 年中開始，我們同時推進越南文、英文、泰文、馬來文的業配偵測。每個語言都需要獨立的 EDA 和標註數據。

**2023 年 9 月 14 日，六語言全部上線**——zh、ja、en、vi、th、ms。這是一個重要的里程碑，代表 TextSponsor 從「中文工具」變成了「多語言系統」。

### 平台擴展（2024）

語言搞定後，開始擴展平台。YouTube、Twitter、TikTok 陸續加入。

YouTube 比較特別——用的是 title field 而不是完整 description，因為 YT description 的格式太不統一，title 反而是更可靠的業配信號。上線時還需要 6 個月的 back-fill，把歷史資料補齊。

---

## 在產品中的角色

TextSponsor 不是一個獨立存在的功能。它是 KOL Radar 數據管線裡的一個節點，下游影響很廣：


**ALS 推薦系統**：TextSponsor 偵測到的業配關係（哪個 KOL 幫哪個品牌發了業配）被餵進 ALS（Alternating Least Squares）訓練，產出品牌推薦。如果一個 KOL 過去常接美妝品牌的業配，系統就會推薦他給其他美妝品牌。

**TextKnowledge**：TextSponsor 判斷「這是業配」之後，TextKnowledge 接手萃取具體的品牌名和商品名。兩者是管線關係——先判斷是不是業配（TextSponsor），再從業配貼文中抽取品牌資訊（TextKnowledge / SPDA）。

**KOL Radar 產品端**：業配偵測結果最終流到產品的 KOL 詳情頁（KD 頁面），呈現為「業配歷史」和「品牌篩選」功能。品牌客戶可以看到一個 KOL 過去跟哪些品牌合作過，幫助判斷是否適合自己。

資料流路徑：TextSponsor → ml_datahub → BigQuery → 產品端分析。

---

## 六語言的挑戰

### 標註數據

每個語言都需要自己的標註數據。不能簡單翻譯——因為不同文化的業配表達方式不同。台灣 KOL 可能寫「感謝 XX 品牌邀約體驗」，日本 KOL 寫「PR」，泰國 KOL 的表達方式又不一樣。

### EDA 的重要性


在訓練模型之前，每個語言都做了獨立的 EDA。目的是理解該語言的業配文本有什麼特徵——常用詞、標記習慣、文本長度分佈。

日文的 EDA 特別重要，因為日文社群的業配標記規範跟其他市場不同。日本有比較明確的「ステマ規制」（隱性業配規範），很多 KOL 會主動標記 PR，這反而讓偵測變得比較容易。

### 語言偵測前置

多語言系統需要先知道貼文是什麼語言，才能送到對應的分類器。這部分由 TextLangID（基於 fastText 的語言偵測服務）負責。整個 NLP 服務群的管線是：TextLangID → TextSponsor → TextKnowledge。

---

## SPDA：從「是不是」到「是誰的」

TextSponsor 回答的是二元問題：是不是業配。但品牌客戶通常更想知道的是：這個業配是哪個品牌的？推了什麼商品？

這就是 SPDA（Sponsor Post Detection & Attribution）的角色。SPDA 在 TextSponsor 之後，做更深層的結構化萃取——從業配貼文中抽取品牌名稱和商品名稱。

```
TextSponsor (是否業配)
    ↓
SPDA (歸因到哪個品牌)
    ↓
TextKnowledge (品牌-商品知識萃取)
```

這個三層管線讓 KOL Radar 可以回答：「這個 KOL 在過去六個月幫哪些品牌做了業配、推了什麼產品」。

---

## 產品化的細節

技術模型做完只是一半。產品化還需要處理幾件事：

**Back-fill**：新語言或新平台上線時，需要把歷史貼文重新跑一次模型。YouTube 上線時做了 6 個月的 back-fill。

**資料更新頻率**：業配偵測不是跑一次就好。新貼文持續進來，模型需要定期跑。這部分由 Datahub 的 ETL pipeline 排程。

**與 K-Score 的關係**：K-Score 是 KOL Radar 的 AI 影響力綜合分數，其中一個因子就是業配頻率。TextSponsor 的準確度直接影響 K-Score 的品質。

**2022 年 11 月 K-Score + Sponsored Post 偵測 BETA 上線**，2023 年 2 月正式公開發布（TW + JP Premium / Enterprise 方案）。


---

TextSponsor 是 AI Lab 最早產品化的模型之一。從一個語言到六個語言、從兩個平台到五個平台，花了大約兩年。

回頭看，我認為最關鍵的決策是**先做繁體中文做到穩定，再擴展語言**，而不是一開始就追求多語言。每個語言的業配表達方式差異太大，如果一開始就想做六語言，會在標註數據和 EDA 上耗太多時間，反而延遲了第一個可用版本的上線。

我還在想的一個問題是：隨著 LLM 能力提升，TextSponsor 這種專門訓練的分類器還有多少存在價值？用 GPT-4 直接判斷一篇貼文是不是業配，效果可能已經不比專門模型差。但成本呢？用 LLM 跑每一篇進來的貼文，跟用一個小模型跑，成本差距可能是百倍。在什麼規模下，切換到 LLM 才划算？

---

*Dennies Hong — AI/ML Product Manager / 寫技術、寫產品、寫真實經歷。*
