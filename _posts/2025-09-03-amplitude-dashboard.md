---
layout: post
title: "從 144M 事件到互動式儀表板：SQLite Pipeline → Next.js 遷移"
permalink: /amplitude-dashboard/
date: 2025-09-03
categories: [數據驅動決策]
tags: [Data Analysis, SaaS, Product Management, Analytics]
author: Dennies Hong
---

這是我推動內部數據自動化的其中一條分支。18,724 個壓縮檔、144M 筆事件、27,898 個工作區。我一個人用 Python + SQLite 做了一條 Pipeline。

KolRadar 用 Amplitude 追蹤用戶行為。每個用戶的每次搜尋、每次點擊、每次收藏都會觸發事件。累積下來就是 8.9GB 的 `.json.gz` 檔案、144 百萬筆事件。

這些數據一直躺在那裡。Amplitude UI 可以看單一指標的趨勢，但跨客戶比較、客戶健康評分、漏斗分析這些事，Amplitude 做不到。得把原始數據拉出來自己算。

## SQLite 為什麼夠用

一開始選 SQLite 純粹是因為不想架任何東西。不需要 server、不需要 Docker、不需要帳號密碼。`pip install` 什麼都不用，Python 內建就有 `sqlite3`。

結果 SQLite 的效能比預期好很多。WAL mode 開下去，144M 筆事件的 ingest 大約 15 分鐘（886 秒）。日級聚合表、月級指標快取、事件明細表全部塞進一個 `kolradar.db` 檔案。

pipeline.py 大約 1,740 行，CLI 介面：

```bash
python pipeline.py ingest            # 18,724 files → 144M events
python pipeline.py compute --force   # 全量計算指標
python pipeline.py export            # 27,913 companies → data/*.json
python pipeline.py ingest-crm        # 匯入 CRM 479 客戶 + 292 付款
python pipeline.py export-benchmark  # benchmark.json
```


<div class="mermaid">
graph TD
    A[18,724 .json.gz<br/>144M 事件] -->|ingest| B[SQLite<br/>WAL mode, 886秒]
    B -->|compute| C[日級/月級聚合表<br/>意圖指標快取]
    C -->|export| D[27,913 company JSON]
    E[CRM Google Sheet<br/>479客戶+292付款] -->|ingest-crm| B
    C -->|export-benchmark| F[benchmark.json<br/>781天×12指標×2組]
    D --> G[Dashboard v1<br/>2,940行 HTML]
    G -->|遷移| H[Dashboard v3<br/>Next.js + Neon PG + Vercel]
    style A fill:#94a3b8,color:#fff
    style B fill:#3b82f6,color:#fff
    style C fill:#6366f1,color:#fff
    style D fill:#8b5cf6,color:#fff
    style G fill:#f59e0b,color:#fff
    style H fill:#10b981,color:#fff
</div>

增量模式靠 `processed_files` 表追蹤已處理的檔案。新數據進來只需要處理差量，不用重跑全部。`_migrate()` 函式處理 schema 變更——直接 `ALTER TABLE` 加欄位，不搞 migration 工具。

## Dashboard v1：2,940 行 HTML

第一版 Dashboard 是一個純 HTML 檔案，Chart.js 畫圖，所有邏輯寫在 `<script>` 裡。


聽起來很粗糙，但它有幾個好處：
- Self-contained，一個檔案搞定，可以直接用 email 寄給業務
- 不需要 build step、不需要 deploy
- Chart.js 4.4.1 從 CDN 載入，其他零依賴

Dashboard 從 `data/{ws_id}.json` 動態載入公司數據，支援多公司切換。七個 Tab：概覽、功能分析、搜尋品質、行為深度、風險監測、趨勢深度、客戶資訊。

### 數據架構

四個 JSON 檔案驅動整個 Dashboard：

1. **company_index.json**——27,913 家公司的索引，含 `has_events` 和 `has_crm` flags
2. **benchmark.json**——781 天 × 12 指標 × 2 組（全體/CRM），0.3MB
3. **qoq_dynamics.json**——季度動態預警矩陣
4. **data/{ws_id}.json**——單一公司的完整數據包

每家公司的 JSON 包含 meta（基本資訊、合約狀態）、months（時間軸）、totals（月度事件量）、users（月度活躍用戶）、categories（21 類別月度數據）、intent_data（10 個意圖指標）、intent_v2（搜尋品質、健康評分等衍生指標）、payments（付款紀錄）、crm（CRM 資料）。

### Benchmark 設計

單一客戶的數字沒有意義，需要 benchmark。兩條基準線：

- **全體平均**：~6,540 個 ws（排除 21,366 個 < 1K events 的試用帳號，佔事件量僅 4.5%，加上 13 個內部帳號）
- **CRM 平均**：464 個有簽約的 ws

Benchmark 從 `daily_event_detail` 聚合，跑 13 秒。前端用虛線疊在客戶圖表上。ECTR 這類比率 benchmark 不能直接取，要用 `bmRatio(click_result, search_intents)` 在 JS 端即時計算。

## 從 HTML 到 Next.js

v2 做到 2,940 行的時候，事情開始失控。

全局總覽的三群組對比（CRM / 非CRM / 排除）、本期快報的週月季切換、結構分析的五個元件、公司詳情的七個 Tab——全部擠在一個 HTML 裡。Chart.js 實例超過 20 個，每次切 Tab 要先 `destroyAllCharts()` 把前一個 Tab 的所有圖表實例清掉，否則會 memory leak。

非 CRM 平均值還要在前端反算：`(all_avg × all_ws - crm_avg × crm_ws) / (all_ws - crm_ws)`。每次日期範圍改變就重算一次，邏輯散落在各個 render 函式裡。


<div class="mermaid">
graph LR
    subgraph v1-v2 單檔架構
        HTML[2,940行 HTML]
        CJS[Chart.js CDN]
        JSON[data/*.json]
    end
    subgraph v3 現代架構
        NJ[Next.js 16 + React 19]
        NE[Neon PG 512MB]
        VE[Vercel 自動部署]
        PR[Prisma 7.6]
        AU[NextAuth v5 OAuth]
    end
    HTML -->|遷移| NJ
    JSON -->|推送| NE
    NJ --> VE
    NJ --> PR
    PR --> NE
    style HTML fill:#f59e0b,color:#fff
    style NJ fill:#10b981,color:#fff
    style NE fill:#3b82f6,color:#fff
    style VE fill:#000,color:#fff
</div>

v3 遷移到 Next.js 16 + Neon PostgreSQL + Vercel。技術棧：
- Next.js 16.2.2（App Router）+ React 19 + TypeScript strict
- Tailwind CSS v4 + Chart.js 4.5.1 + react-chartjs-2
- Prisma 7.6.0 + Neon PostgreSQL（free tier，512MB）
- NextAuth v5 + Google OAuth 白名單

### Neon 512MB 的限制

Free tier 只有 512MB，144M 筆原始事件塞不進去。所以只推聚合數據：

| 表 | 筆數 |
|---|---|
| workspaces | 28,760 |
| monthly_kpi | 54,867 |
| crm_data | 467 |
| payments | 292→754 |
| benchmark | 456 |
| market_segments | 49 |

後來加了 Phase B 的五張每日聚合表（只保留最近 6 個月）：`kol_master`（538 KOL）、`daily_kol_interaction`（568K rows）、`daily_search_keyword`（61K rows）、`daily_collection_usage`、`daily_campaign_usage`、`daily_report_usage`。

monthly_sub_events 從 229MB 清到 87MB，只留最近半年。後來又因為新功能加回來到 245MB。一直在跟 512MB 的限制搏鬥。

### Route 結構

```
/dashboard           → 全局總覽（本期快報 + 結構分析）
/company             → 公司列表（搜尋/篩選/排序/分頁）
/company/[wsId]      → 概覽（健康儀表+風險+漏斗+意圖+趨勢）
/company/[wsId]/features        → 功能分析
/company/[wsId]/search-quality  → 搜尋品質
/company/[wsId]/behavior        → 行為深度
/company/[wsId]/risk            → 風險監測
/company/[wsId]/trends          → 趨勢深度
/company/[wsId]/info            → 客戶資訊
/company/[wsId]/details         → 客戶細節（KOL/關鍵字/收藏/商案/報告）
/company/[wsId]/intent          → 使用意圖分析
/metrics             → 指標列表（功能+品質+子事件）
/metrics/[metricKey] → 指標詳情（6 區塊）
/company-groups      → 公司群組（22,642 家，ws 聚合）
```

每個 route 都是 Server Component 做首次渲染，搜尋篩選透過 API route 走 Prisma 查 Neon。

### 風險監測改版

原本的風險偵測用固定門檻（比如月事件量 < 100 就標紅）。但不同規模的客戶，「正常」的門檻完全不同。

改成跟自己比：用前三個月的平均值作為基線，跟當月比較。超過基線 2 倍標 spike，低於基線 50% 標 warning，低於 30% 標 critical。同一套邏輯套到 AI 搜尋棄用率和 KD 頁瀏覽尖峰。

## 踩坑紀實

幾個印象深刻的坑：

- **Node 版本**：Prisma 在 Node 21 會噴 `ERR_REQUIRE_ESM`，必須用 Node 22。但專案沒有 `.nvmrc`，每次換台機器都要記得切版本


- **XSS**：ws:67970 的 org 欄位含有 `<img src=x onerror=alert(1)>`。一開始用 `innerHTML` 直接輸出，後來全面加了 `esc()` 函式跳脫 HTML

- **GSheet CSV 前兩行**：CRM 資料從 Google Sheet 匯出的 CSV，前兩行是篩選條件，第三行才是 header。Pipeline 要知道 skip 前兩行

- **Benchmark 垃圾日期**：沒加日期過濾的話，benchmark export 會出現 1970 年的資料。需要 `>= 2024-01-01` 才能確保乾淨

- **workspaceName 不是公司名**：Amplitude 的 `user_properties.workspaceName` 是工作區名稱，不是公司名。要用 `company` 欄位。搞混了會把同一家公司的不同工作區拆成不同公司

- **Workspace 名稱三層補齊**：CRM 只覆蓋 464 家，Admin 後台加 18,105 家，Amplitude events 的 company 欄位再加 3,934 家。最終覆蓋 22,503/27,913（80.6%），剩下 5,410 標記為（未知），全部 < 1K events

## 現在的狀態

v3 已經部署在 Vercel，push main 自動部署。Google OAuth 白名單控制存取。五位測試使用者可以登入。

全局總覽有本期快報（週/月/季報）和結構分析（趨勢、市場比較、指標 sparkline、QoQ 動態、Top 20）。公司詳情有 9 個 Tab。指標列表有 19 個功能指標、12 個品質指標、362 個子事件。

Pipeline 還是跑在本地的 SQLite 上，每次有新數據就手動 `ingest → compute → push-pg`。沒有接 Amplitude 的 export API 做自動化，因為 Amplitude 的 export API 有 quota 限制，每次拉全量也太慢。

---

我認為這個專案最大的 tradeoff 是「一個人能做到什麼程度」。沒有後端工程師幫我架 data pipeline，沒有前端工程師幫我寫 Dashboard，沒有 DevOps 幫我搞 CI/CD。所以選了最低維護成本的技術棧：SQLite 不需要 server，HTML 不需要 build，Neon free tier 不需要費用，Vercel 不需要設定。

但代價是：2,940 行的單一 HTML 最後還是撐不住，得花時間遷移到 Next.js。如果一開始就用 Next.js，可能不會走這條彎路。不過如果一開始就用 Next.js，可能前兩週就會花在搞 build 和 deploy 上面，而不是在分析數據。

還沒解決的問題：512MB 的 Neon free tier 隨時可能爆。目前的做法是只留最近 6 個月的明細數據，但如果要做同比分析（今年 Q1 vs 去年 Q1），就需要保留更長的歷史。升級到付費方案是一個選項，但這也意味著這個 side project 開始產生 recurring cost。

---

*Dennies Hong — AI/ML Product Manager / 寫技術、寫產品、寫真實經歷。*
