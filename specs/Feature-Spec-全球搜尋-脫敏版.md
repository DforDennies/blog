---
layout: page
title: "Feature Spec — 全球網紅搜尋"
permalink: /specs/Feature-Spec-全球搜尋-脫敏版/
---

> **產品**：KOL SaaS 平台  
> **類型**：Feature Specification  
> **日期**：2024-07  
> **狀態**：Shipped

---

## Problem Alignment

### Brief

世界各國的訪客會透過自己已知的網紅社群帳號來測試平台：

1. 能不能找到這個網紅？
2. 能不能分析這個帳號？

如果連該地區的活躍網紅都搜不到，用戶不會繼續試用。這直接影響國際市場拓展的評估 — Product Team 需要量化判斷是否值得進入某個市場。

### Goals

讓使用者透過 Quick Search 找到**世界上任一網紅**，即使該網紅尚未存在於平台資料庫中。

### 影響範圍確認

| 維度 | 範圍 |
|------|------|
| 產品介面 | App、Landing Page、Admin、Chrome Extension |
| 社群平台 | Instagram、YouTube、Facebook、TikTok、X |
| 商業影響 | 不影響訂閱/購買流程 |

---

## Solution Alignment

### Acceptance Criteria

功能部署於 **App / Landing / Admin / Chrome Extension** 四個入口。

**Quick Search 調整**：
- 輸入網紅名稱或帳號時，即時查詢外部資料源
- 若平台無此網紅，從第三方 API 即時取得基本資料並回傳
- 搜尋結果標示「新網紅」標記，區分已建檔 vs 即時查詢

**搜尋結果頁調整**：
- 即時取得的新網紅顯示可用數據（粉絲數、簡介、近期內容）
- 提供「更新完通知我」按鈕，用戶可訂閱完整數據更新通知

### Flow

<div class="mermaid">
graph TD
    A[用戶輸入搜尋] --> B{平台資料庫<br/>有此網紅？}
    B -->|有| C[顯示完整 KOL 資料]
    B -->|沒有| D[查詢第三方 API]
    D --> E{API 有資料？}
    E -->|有| F[顯示基本資料<br/>標記「新網紅」]
    E -->|沒有| G[顯示無結果]
    F --> H[用戶點擊進入 KD 頁]
    F --> I[用戶點擊<br/>「更新完通知我」]
    I --> J[系統排程完整爬取]
    J --> K[爬取完成<br/>發送通知]
    
    style B fill:#fff3e0,stroke:#e65100
    style F fill:#e3f2fd,stroke:#1565c0
    style I fill:#e8f5e9,stroke:#2e7d32
</div>

---

## Tracking（埋點設計）

### Event 1：Quick Search for KOL

> 觸發時機：用戶執行搜尋，取得結果

| 屬性 | 類型 | 說明 |
|------|------|------|
| `kolResult` | Array\<ID\> | 搜尋結果的 KOL ID 列表（既有屬性） |
| `kolNames` | Array\<String\> | 搜尋到的 KOL 名稱 |
| `isNewToPlatforms` | Array\<Boolean\> | 是否為平台新網紅（`true` = 即時查詢，`false` = 已建檔） |
| `dataSources` | Array\<String\> | 資料來源標記（已建檔者為 `DC`） |

### Event 2：Choose Search Result

> 觸發時機：用戶點擊某個搜尋結果

| 屬性 | 類型 | 說明 |
|------|------|------|
| `kolId` | String | KOL ID（新網紅不傳） |
| `kolDcId` | String | DC ID（新網紅不傳） |
| `kolName` | String | KOL 名稱 |
| `isNewToPlatform` | Boolean | 是否為新網紅 |
| `dataSource` | String | 資料來源 |

### Event 3：Visit KOL Detail

> 觸發時機：用戶進入 KOL 詳細頁

| 屬性 | 類型 | 說明 |
|------|------|------|
| `path` | String | 頁面路徑 |
| `from` | String | 來源頁面 |
| `kolName` | String | KOL 名稱 |
| `kolCountry` | String | KOL 國家 |
| `isNewToPlatform` | Boolean | 是否為新網紅 |
| `dataSource` | String | 資料來源 |

### Event 4：Subscribe KOL Data Update Notification

> 觸發時機：用戶點擊「更新完通知我」按鈕  
> 狀態：因觸發條件限制，暫停開發

---

## 分析目標

1. 有多少用戶點擊「更新完通知我」？
2. 進入 KD 頁的新舊 KOL 數量差異
3. 點擊按鈕的轉化率，可依資料來源切分
4. 主要觸發的資料來源分佈

---

*Dennies Hong — AI/ML Product Manager / 寫技術、寫產品、寫真實經歷。*
