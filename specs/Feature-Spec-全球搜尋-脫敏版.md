---
layout: page
title: "可以搜尋到全世界的網紅"
permalink: /specs/Feature-Spec-全球搜尋-脫敏版/
---

# 可以搜尋到全世界的網紅

> 產品：Platform K（KOL SaaS）  
> 類型：Feature Spec  
> 建立：2024-07-16  
> 最後編輯：2026-02-19

---

# Problem Alignment

## Brief

  世界各國造訪者會透過自己已知的網紅社群帳號，來測試這個平台

  1. 是否至少可以找到
  1. 且能分析該帳號
  來判斷是否繼續用用看。如果連該活躍地區的網紅都找不到，也不會有其他試用。

  這對於 country management 中 product team 想透過量化協助判斷是否進入單一市場，會是一個問題。

---

## Goals & Success

- 讓使用者可以透過 quick search 找到世界上任一網紅，就算他目前不在平台資料庫內
Should we... 

- [ ] 是否有要新版介紹 tutorial
- [ ] 確認影響範圍有哪些國家 workspace 或語言
- [ ] 確認影響範圍有哪些平台，Landing、App、Chrome Extension、Admin
- [ ] 確認影響範圍有哪些社群平台，IG, YT, FB, TikTok, X
- [ ] 確認是否影響到線上訂閱或線下購買
- [ ] 是否有寫埋點與確認用字
---

# Solution Alignment

## Acceptance Criteria

We empower features in **App / Landing / Admin / Chrome.** If you find something should also be in another place, please remind the PM for a better delivery quality. 

## Quick Search 的調整

## 一般搜尋結果的調整

更新回來要有哪些數據與資料

<details><summary>請參考以下，和 Chrome extension 以及搜尋完一樣（自動建立收到回報的未建立網紅）</summary>

  （詳見內部文件）

</details>



---

## Flow / Wireframe

[Wireframe: Global Influencer Retrieval]

---

## Mockup / Prototype

[Mockup]

---

## Tracking

- 搜尋到 KOL 時，影響事件 `**Quick Search for KOL**`
  - 影響既有屬性
    - **kolResult : 這是 kolId 的 Array**
  - 新增屬性
    - kolNames : 搜尋到的 kol 名稱，Array
    - isNewToPlatforms : Array, 內容為布林，這如果是沒資料新 KOL 傳 True，不是的話傳 False ( 之後有資料就會是 False )
    - dataSources : Array，內容為 String，記錄資料從哪來，KOL 有資料後這值會變 DC


- 點擊 KOL 時，影響事件 `**Choose Search Result**`
  - 影響既有屬性
    - kolId : 新的 KOL 不傳
    - kolDcId : 新的 KOL 不傳
  - 新增屬性
    - kolName : 搜尋到的 kol 名稱，String
    - isNewToPlatform : 布林，這如果是沒資料新 KOL 傳 True，不是的話傳 False ( 之後有資料就會是 False )
    - dataSource : String，記錄資料從哪來


- 進入 KD 頁時，影響事件 `**Visit KOL Detail**`
  - 只傳這些屬性就好
    - path
    - from
    - isDefault
    - keyword
    - kolName
    - kolCountry
  - 新增屬性
    - isNewToPlatform : 布林，這如果是沒資料新 KOL 傳 True，不是的話傳 False ( 之後有資料就會是 False )
    - dataSource : String，記錄資料從哪來
    

- 點擊更新完通知我，新增事件記錄
  - 事件名稱 : `**Subscribe KOL Data Update Notification**`
    - 屬性（詳見 Amplitude 埋點定義）


### Event Automation Test

- `**Quick Search for KOL**`
  - b4aaa5b5967e4728cd0be47c7e78c6f8bfb99d15
- `**Choose Search Result**`
  - 3198693101ce3d897b77bcdaa82c77b9744a5fa6
- `**Visit KOL Detail**`
  - 283c36b5024be73e4e426fc3afc216048e1722a4
- `**Subscribe KOL Data Update Notification**`
  - TBD (網紅被觸發事件後，無法重複使用，故先暫停開發)

---

## 分析目標

1. 有多少人點擊按鈕
1. 進入 KD 新舊 KOL 數量差異
1. 點按鈕的轉化率，還可以切不同資料源
1. 點按鈕的主要資料來源

---

# Reference

- 競品費用評估文件
- 內部討論文件
---
