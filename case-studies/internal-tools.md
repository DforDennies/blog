---
layout: page
title: "Case Study: Internal Tools — 沒有工具？我自己做"
permalink: /case-studies/internal-tools/
---

2024 年初，公司裁員後 Data 團隊歸零。沒有 BI 工具、沒有 data pipeline、沒有任何一份可以信任的報表。

銷售數字散落在三個 Google Sheet 裡，格式不統一。GA4 的 97% Key Events 是 scroll，所有轉換率都是假的。Amplitude 買了但沒人打開。業務問「這個客戶下個月會不會續約？」——CRM 上面只寫著「合約中」，回答不了。

我是 AI/ML PM，不是 Data Engineer。但看不到全景就不可能推動任何事。所以我自己搭了三套數據系統。

---

## 問題

裁員之後，能回答「我們的數據怎麼樣」的人只剩我一個。具體來說：

- **營收數據不可信**：BQ Pipeline 算出來的金額跟業務報表差了一百多萬。同一家客戶在三個系統裡叫三個不同名字
- **流量歸因全盲**：GA4 裡 100% 的 sessions 歸為 Unassigned。行銷花了預算辦活動、投廣告，但沒人知道哪個管道有效
- **客戶健康不可見**：144M 筆行為事件躺在 Amplitude 裡，沒有人把它變成業務能用的東西。客戶是在用產品還是準備流失？不知道

三個問題，三套工具。從零開始，一個人做。

---

## 三個自建工具

### 1. 銷售報表 Pipeline

**為什麼要做**：業務手填的 Google Sheet、後台訂閱系統、財務發票紀錄——三個數字對不上。年度差距 $1.39M。沒有乾淨的營收數據，GRR、NRR 這些 SaaS 核心指標全部算不出來。

**怎麼做的**：Python + BigQuery + gspread，五層配對邏輯從最嚴格到最寬鬆逐層 fallback：

<div class="mermaid">
graph TD
    A["BQ Pipeline 交易紀錄"] --> M{"五層配對引擎"}
    B["業務 Google Sheet"] --> M
    M --> L1["Layer 1: Exact Match<br/>品牌+月份完全一致"]
    L1 -->|未配對| L2["Layer 2: Alias Match<br/>101 組中英異名"]
    L2 -->|未配對| L3["Layer 3: Normalized<br/>移除公司後綴"]
    L3 -->|未配對| L4["Layer 4: ±1 Month<br/>時間基準差異"]
    L4 -->|未配對| L5["Layer 5: Purchase Split<br/>底線拆分+substring"]
    L5 --> R["配對率 60% → 99.6%"]
    style A fill:#3b82f6,color:#fff
    style B fill:#f59e0b,color:#fff
    style R fill:#10b981,color:#fff
</div>

五層配對之外，還有九大修正機制：品牌別名 101 組、金額差異 22 筆、跳過除稅 13 筆、月份偏移 35 筆、NW/OD 覆蓋 127 筆、BQ 缺漏注入 156 筆。合計約 340 筆 config，每一筆都是跟業務逐一確認後才加的。

**成果**：

| 指標 | 數值 |
|------|------|
| 追蹤交易數 | 1,133 筆 / 593 家客戶 / 8 個市場 |
| 金額配對率 | 800/803 (99.6%) |
| NW/OD 配對率 | 748/748 (100%) |
| 年度差距 | $1.39M → $24K |
| 輸出 | 9 Tab Google Sheet，每日自動更新 |

拿這套乾淨數據去算 GRR，結果是 40.50%。數字不漂亮，但至少是一個可信的數字。之前沒人敢相信報表，因為兩邊對不上。

---

### 2. Amplitude 互動式儀表板

**為什麼要做**：Amplitude UI 可以看單一指標趨勢，但跨客戶比較、健康評分、漏斗分析做不到。業務需要的是「這個客戶健不健康」，不是「這個事件觸發了幾次」。

**怎麼做的**：18,724 個 .json.gz 壓縮檔 → Python + SQLite 增量 ETL → 月級聚合 → JSON export → Dashboard。

<div class="mermaid">
graph LR
    A["18,724 .json.gz<br/>144M 事件 / 8.9GB"] -->|"Python ETL<br/>886 秒"| B["SQLite<br/>日級/月級聚合"]
    B -->|"compute"| C["486 事件 → 21 類<br/>10 個意圖指標"]
    C -->|"export"| D["27,913 家<br/>公司 JSON"]
    D --> E["Dashboard v1<br/>2,940 行 HTML"]
    E -->|"遷移"| F["Dashboard v3<br/>Next.js + Neon PG"]
    G["CRM 479 客戶"] -->|"ingest"| B
    style A fill:#94a3b8,color:#fff
    style B fill:#3b82f6,color:#fff
    style C fill:#6366f1,color:#fff
    style F fill:#10b981,color:#fff
</div>

核心設計——八維健康評分（滿分 100）：

| 維度 | 權重 | 衡量什麼 |
|------|------|----------|
| 搜尋活躍 | 20% | 有沒有在找網紅 |
| 漏斗深度 | 15% | 搜尋→收藏→商案走多深 |
| 搜尋品質 | 15% | ECTR，搜完有沒有點 |
| 團隊規模 | 10% | 多少人在用 |
| 穩定度 | 10% | 活躍天數是否穩定 |
| 成長趨勢 | 10% | MoM 方向 |
| 功能採用 | 10% | 21 類用了幾類 |
| 行為成熟 | 10% | 工作流成熟度 |

兩條 Benchmark 基準線：全體平均（~6,540 ws）和 CRM 平均（464 簽約客戶）。單一客戶的數字沒有意義，要跟同類比才知道好壞。

**成果**：

- 覆蓋 27,898 個工作區、22,642 家公司（Union-Find + LLM 歸戶）
- 9 個分析 Tab：概覽、功能分析、搜尋品質、行為深度、風險監測、趨勢、客戶資訊、細節、使用意圖
- 發現「行為成熟度」27 個月只從 0.1 升到 0.2——有人用，但沒人用得深
- 部署在 Vercel，Google OAuth 白名單控制存取

---

### 3. GA4 流量歸因框架

**為什麼要做**：老闆問 Paid Social 值不值得。打開 GA4，97% Key Event 是 scroll。所有管道轉換率都被灌到 50% 以上。這等於兩年來看過轉換率報表的人，看到的都是假數字。

**怎麼做的**：排除假事件（scroll / Amplitude 同步 / Audience trigger），重新定義 9 個有意義的 Key Events，從 BigQuery Export 拉原始數據做全量分析。

9 個 Key Events 按漏斗層級：
- **漏斗上層**：View Signup Screen (~6,149/月)、Login Success (~3,085/月)
- **核心轉換**：Account Created AD (~629/月)、Account Created KOL (~60/月)、Clicks Register (~581/月)
- **高價值 Lead**：Complete Premium Trial (~208/月)、Submit Contact-Us (~35/月)
- **內容轉換**：Download Success (~43/月)

**成果**：

| 管道 | 修正前轉換率 | 修正後轉換率 | 判定 |
|------|------------|------------|------|
| Email | 52.5% | 18.8% | S 級，被低估 |
| Organic Social | 51.2% | 12.0% | S 級，值得加量 |
| Paid Search | 43.4% | 6.2% | A 級 |
| **Paid Social** | **3.0%** | **0.3%** | **C 級，ROI 偏低** |
| **Display** | **1.2%** | **0.1%** | **C 級，ROI 偏低** |

挖出四個緊急問題：39% Landing Page 是 `(not set)`、$190K 收入全歸 Unassigned、Direct 佔 30.6% 疑似 UTM 缺失、SG/TH 異常流量增長（疑似 bot）。

最終框架涵蓋 27 個月數據、163 個行銷活動追蹤、五層報告系統（日/週/月/季/年），含歷史回溯機制。

---

## 技術選擇

每個工具的技術棧都是同一個原則選的：**一個人能維護、不需要公司資源、今天就能用**。

| 工具 | 技術棧 | 為什麼這樣選 |
|------|--------|-------------|
| 銷售報表 | Python + BigQuery + gspread | BQ 已有交易數據，gspread 直接推 Google Sheet，業務零學習成本 |
| Amplitude 儀表板 | Python + SQLite → Next.js 16 + Neon PG + Vercel | SQLite 不需要 server、Python 內建。遷移是因為 2,940 行 HTML 撐不住 |
| GA4 歸因 | Python + BigQuery Export + HTML 報告 | GA4 每天自動匯入 BQ，2,126+ 張日表，等於 raw data 完全控制權 |

**為什麼 SQLite → Neon PG**：v1 用 SQLite 是因為不想架任何東西。`pip install` 什麼都不用，Python 內建就有 `sqlite3`。但做到 2,940 行 HTML、20+ Chart.js 實例、切 Tab 要先 `destroyAllCharts()` 清 memory leak 的時候，就知道該遷了。Neon free tier 512MB 剛好夠放聚合數據，配 Prisma + Next.js App Router，每個 Tab 變獨立 route。

**為什麼不用 Airflow / dbt / Metabase**：一個人用不起這些工具的維護成本。1,740 行的 `pipeline.py` 用 CLI 操作（`ingest → compute → export`），比設定 Airflow DAG 快十倍。

---

## 成果

<div class="mermaid">
graph TD
    subgraph "三套 Pipeline 交叉驗證"
    S["銷售報表<br/>1,133 筆交易<br/>GRR 40.5%"]
    G["GA4 流量歸因<br/>27 個月 × 12 管道<br/>163 個活動"]
    A["Amplitude 行為<br/>144M 事件<br/>8 維健康評分"]
    end
    S --> F["完整漏斗首次可見"]
    G --> F
    A --> F
    F --> I1["頂端：98% 進站流量<br/>沒看到註冊頁"]
    F --> I2["中段：搜尋→收藏<br/>掉 93%"]
    F --> I3["底端：79% 客戶<br/>只買一次就走"]
    style S fill:#3b82f6,color:#fff
    style G fill:#f59e0b,color:#fff
    style A fill:#8b5cf6,color:#fff
    style I1 fill:#ef4444,color:#fff
    style I2 fill:#f97316,color:#fff
    style I3 fill:#ef4444,color:#fff
</div>

三套工具疊起來，公司第一次能同時看到：

- **銷售**：誰簽了約、多少錢、什麼方案、誰負責、GRR 多少
- **流量**：從哪來、走到哪裡、在哪裡斷掉、哪個管道有效
- **行為**：簽約之後用了什麼、用多深、什麼時候開始不用

具體數字：
- 銷售報表年度差距從 **$1.39M 壓到 $24K**，金額配對率 99.6%
- GA4 發現 Paid Social 真實轉換率 **0.3%**（修正前顯示 3.0%），Display **0.1%**
- Amplitude 追蹤 **27,898 個工作區**，八維健康評分覆蓋所有簽約客戶
- 三套報表共追蹤 **27 個月**數據，含歷史回溯

最關鍵的發現：三個視角指向同一個問題——**漏斗每一段都在漏**。頂端 98% 流量沒看到註冊頁，中段搜尋到收藏掉 93%，底端 79% 客戶只買一次就走。全期間自然續約率平均只有 2%。

---

## 關鍵學習

**PM who builds > PM who specs。** 這三套工具沒有一套是排進 sprint 的。沒有 PRD、沒有設計稿、沒有工程資源。如果我只會寫 spec 然後等人做，這些工具到現在都不會存在。

**選最低維護成本的技術棧。** SQLite 不需要 server，HTML 不需要 build，Neon free tier 不需要費用，Vercel 不需要設定。一個人的 side project 最怕的不是功能不夠，是維護成本把你壓垮。

**先有數據，再有決策。** 在這三套系統存在之前，行銷看流量但不看轉換、業務看簽約但不看續約、產品看功能但不看付費行為。沒有人串起來看。工具的價值不只是「有圖表可以看」，而是讓散落在三個系統裡的數據第一次講同一個故事。

**340 筆 config 的耐心。** 銷售報表的 340 筆修正項，每一筆都是跟業務逐一確認後才加的。品牌別名、金額差異、月份偏移、合約拆期——這些不是寫程式能解決的，是坐下來一筆一筆對出來的。技術能力讓你建工具，但讓工具真正有用的是對業務細節的耐心。

---

## 相關文章

- [從 144M 事件到互動式儀表板：SQLite Pipeline → Next.js 遷移](/2026/03/18/amplitude-dashboard/)
- [為什麼兩張表比金額需要 340 筆 Config？SaaS 業務數據比對實戰](/2026/03/14/銷售報表-340config/)
- [客戶健康評分：8.9GB 事件 → 21 類月度快照](/2026/03/15/客戶健康評分/)
- [SaaS 流量歸因引擎：9 Key Events 設計邏輯](/2026/03/16/ga4-流量歸因/)
- [我偷偷搭了三套數據系統](/2026/04/06/s1-ep1-三套pipeline的誕生/)
