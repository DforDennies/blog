---
layout: page
title: "Platform K 銷售報表規格書"
permalink: /specs/Data-Spec-銷售報表-脫敏版/
---

> 目標：復刻 Google Sheet「2023-2026銷售紀錄」17 欄 + 擴充三來源獨有資料  
> 資料來源：BQ（[內部資料庫]）+ Admin Panel + Amplitude  
> 主鍵：交易層級（每筆 billing / operation_memo 一列），以 `workspace_id` 串接  
> 資料範圍：2024/01/01 起  
> 運行模式：冷啟動回滾 + 每日增量更新  
> 產出格式：待定（Notion / Google Sheet / CSV）

---

## Part A — Google Sheet 復刻欄位（17 欄）

### 可自動產出（8 欄）

| # | 欄位 | 來源 | 邏輯 |
|---|---|---|---|
| 2 | 品牌 | 線上: `billings.company_name`<br>線下: `workspaces.company` (by workspace_id from operation_memos) | |
| 6 | 進案月份 | 線上: `DATE_FORMAT(billings.checkout_at, '%Y/%m')`<br>線下: `DATE_FORMAT(operation_memos.created_at, '%Y/%m')` | |
| 8 | 方案 | `plans.duration` → 30=月方案, 90=季方案, 180=半年方案, 365=年方案 | |
| 9 | 版本 | `plans.name` 拆解：<br>vip→專業版(舊), premium→專業版(新), advanced→企業版,<br>premium_202408→New Premium, advanced_202408→New Advanced,<br>startup→Startup, corporate→客製版本, trial_202408→試用版 | |
| 11 | 線上線下 | `billings` 有紀錄 → online<br>`operation_memos` reason=購買方案 且 billings 無紀錄 → offline | |
| 15 | 發票狀態 | `payment_transactions.invoice_status`:<br>is_invoiced → Y, not_invoiced → N, init → 未處理 | 透過 `billings.payment_transaction_id` 或 `orders.payment_transaction_id` join |
| 16 | 方案啟用 | 線上: `billings.completed_at`<br>線下: `workspace_auths.started_at` (source_type=operation_memo) | |
| 17 | 方案結束 | `workspace_auths.expire_at` | |

### 部分可用（4 欄）

| # | 欄位 | 來源 | 限制 |
|---|---|---|---|
| 4 | 產業 | `workspaces.industry`（英文 enum 18類） | 需建 mapping table 對應中文；粒度比 Google Sheet 粗 |
| 7 | 進案金額(未稅) | 線上: `billings.sub_total`<br>線下: `operation_memos.value / 1.05`（TWD）或 `/1.1`（JPY） | 線下金額是含稅，需反推；客製折扣案可能不準 |
| 13 | 發票開立月份 | `payment_transactions.updated_at`（當 invoice_status 變為 is_invoiced 的時間） | 只是近似值，非實際開立日 |
| 14 | 發票金額(未稅) | `payment_transactions.invoice_amount`（is_invoiced 有值，共 221 筆/總額 456,535） | 主要是額度購買的發票，訂閱發票可能不在此 |

### 無法自動化（5 欄）→ 需人工維護

| # | 欄位 | 說明 |
|---|---|---|
| 1 | 購買明細(KB編號) | CRM/ERP 內部合約編號，BQ 無此資料 |
| 3 | 電商 | 人工標記是否電商客戶 |
| 5 | 營業項目 | 人工細分（保健食品、汽車等），比 industry 更細 |
| 10 | 新舊客戶 | 人工判斷 OD(舊客)/NW(新客) |
| 12 | Sales 業務 | 業務姓名。BQ sales_code 是 coupon code，operation_memos.admin_id 是操作人員(ops)，都不是業務 |

---

## Part B — 擴充欄位（三來源獨有）

### B1. BQ 獨有（帳務 & 身份）

| 欄位 | 來源 | 說明 |
|---|---|---|
| workspace_id | `billings.workspace_id` / `operation_memos.workspace_id` | 主鍵，串接所有系統 |
| email | `billings.email` | 付款人 email |
| 統編 | `billings.tax_id` | |
| 是否公司 | `billings.is_company` | 1=公司, 0=個人 |
| 含稅金額 | 線上: `billings.amount`<br>線下: `operation_memos.value` | |
| 稅額 | `billings.tax_fee` | |
| 折扣金額 | `billings.discount` | |
| 幣別 | `billings.currency` / `operation_memos.currency` | TWD / JPY / USD |
| coupon_code | `billings.coupon_code` | 使用的優惠碼 |
| 訂單編號 | `orders.order_no` | 格式 TW2023... |
| 取消原因 | `subscription_cases.cancel_reason` | user_cancel 等 |
| 國家 | `workspaces.country_code` | tw/jp/us/hk/my 等 |
| 公司規模 | `workspaces.business_size` | level0~level6 |
| 使用目的 | `workspaces.usage_purpose` | lookingForCandidates 等 |
| 操作人員 | `operation_memos.admin_id` → `admins.name` | 線下開通的 ops 人員 |

### B2. Admin Panel 獨有（合約狀態）

| 欄位 | 來源 | 說明 |
|---|---|---|
| 組織名稱 | `admin.org` | 有時比 BQ company 更準確 |
| 讓利 % | `admin.discount` | 線下折扣比例 |
| 開通服務 | `admin.services` | Directory, Collection, Hashtag, Report 等 |
| feature_flag | `admin.feature_flag` | |
| 額度刷新日 | `admin.quota_refresh` | |
| 座位上限 | `admin.seat_limit` | e.g. 5/5 |
| 測試帳號 | `admin.test_account` | 需排除 |
| 帳號狀態 | `admin.status` | Active / Expired |

### B3. Amplitude 獨有（使用行為）

| 欄位 | 來源 | 說明 |
|---|---|---|
| 活躍用戶 W1~W4 | Amplitude Segmentation API | 最近 4 週各週活躍用戶數 |
| 搜尋 KOL W1~W4 | 同上 | |
| 解鎖 KOL W1~W4 | 同上 | |
| 收藏 KOL W1~W4 | 同上 | |
| 使用趨勢 | 計算欄位 | W4 vs W1 比較：上升 / 下降 / 持平 |

---

## Part C — 資料串接邏輯

### 主查詢（線上交易）
```sql
SELECT
  b.workspace_id,
  b.company_name                              AS 品牌,
  w.industry                                  AS 產業,
  FORMAT_TIMESTAMP('%Y/%m', b.checkout_at)     AS 進案月份,
  b.sub_total                                 AS 進案金額_未稅,
  b.amount                                    AS 含稅金額,
  b.tax_fee                                   AS 稅額,
  b.discount                                  AS 折扣金額,
  b.currency                                  AS 幣別,
  -- 方案
  CASE p.duration
    WHEN 30 THEN '月方案' WHEN 90 THEN '季方案'
    WHEN 180 THEN '半年方案' WHEN 365 THEN '年方案'
  END                                         AS 方案,
  -- 版本
  CASE
    WHEN p.name LIKE 'advanced_202408%' THEN 'New Advanced'
    WHEN p.name LIKE 'premium_202408%' THEN 'New Premium'
    WHEN p.name LIKE 'advanced%' THEN '企業版'
    WHEN p.name LIKE 'premium%' THEN '專業版'
    WHEN p.name LIKE 'vip%' THEN '專業版(舊)'
    WHEN p.name LIKE '%startup%' THEN 'Startup'
    WHEN p.name LIKE '%corporate%' THEN '客製版本'
    ELSE p.name
  END                                         AS 版本,
  'online'                                    AS 線上線下,
  b.sales_code,
  b.coupon_code,
  b.email,
  b.tax_id                                    AS 統編,
  b.is_company,
  -- 發票
  pt.invoice_status                           AS 發票狀態_raw,
  CASE pt.invoice_status
    WHEN 'is_invoiced' THEN 'Y'
    WHEN 'not_invoiced' THEN 'N'
    ELSE '未處理'
  END                                         AS 發票狀態,
  pt.invoice_amount                           AS 發票金額,
  FORMAT_TIMESTAMP('%Y/%m', pt.updated_at)    AS 發票開立月份_近似,
  -- 時間
  b.checkout_at,
  b.completed_at                              AS 方案啟用,
  wa.expire_at                                AS 方案結束,
  -- 串接鍵
  b.workspace_id,
  w.country_code                              AS 國家,
  w.business_size                             AS 公司規模
FROM [內部資料庫].billings b
LEFT JOIN [內部資料庫].plans p
  ON b.plan_id = p.id
LEFT JOIN [內部資料庫].workspaces w
  ON b.workspace_id = w.id
LEFT JOIN [內部資料庫].workspace_auths wa
  ON b.workspace_id = wa.workspace_id
LEFT JOIN [內部資料庫].payment_transactions pt
  ON b.payment_transaction_id = pt.id
WHERE b.status = 'paid'
ORDER BY b.checkout_at DESC
```

### 線下交易查詢
```sql
SELECT
  om.workspace_id,
  w.company                                   AS 品牌,
  w.industry                                  AS 產業,
  FORMAT_TIMESTAMP('%Y/%m', om.created_at)     AS 進案月份,
  CAST(om.value AS FLOAT64) / 1.05            AS 進案金額_未稅_推估,
  CAST(om.value AS INT64)                     AS 含稅金額,
  om.currency                                 AS 幣別,
  'offline'                                   AS 線上線下,
  a.name                                      AS 操作人員,
  om.operation,
  om.reason,
  om.created_at                               AS 方案啟用_近似,
  wa.expire_at                                AS 方案結束,
  -- 串接鍵
  om.workspace_id,
  w.country_code                              AS 國家
FROM [內部資料庫].operation_memos om
JOIN [內部資料庫].admins a
  ON om.admin_id = a.id
LEFT JOIN [內部資料庫].workspaces w
  ON om.workspace_id = w.id
LEFT JOIN [內部資料庫].workspace_auths wa
  ON om.workspace_id = wa.workspace_id
WHERE om.reason = '購買方案'
  AND om.operation = '開通購買權限'
ORDER BY om.created_at DESC
```

### Admin / Amplitude 補充
以 `workspace_id` 為鍵，LEFT JOIN：
- Admin Panel 資料（讓利%, services, feature_flag, 額度刷新日, 座位上限）
- Amplitude 週報資料（活躍用戶, 搜尋/解鎖/收藏 KOL W1~W4, 趨勢）

---

## Part D — 產業 Mapping Table（待建）

| BQ industry (EN) | Google Sheet 對應 | 備註 |
|---|---|---|
| advertisingMarketing | Agency | |
| eCommerce | eCommerce | |
| beauty | Beauty | |
| medicalHealthcare | Pharma / Healthcare | |
| softwareInternet | Tech | |
| entertainment | Entertainment | |
| travelLeisure | Travel | |
| computersElectronics | Electronics | |
| education | Education | |
| otherManufacturing | Manufacturing | |
| financial | Finance | |
| retailWholesale | Retail | |
| foodBeverage | F&B | |
| gaming | Gaming | |
| telecommunications | Telecom | |
| govNonProfit | Gov/NPO | |
| realEstate | Real Estate | |
| others | Others | |

---

## Part E — 排除條件

- `workspaces.is_test = 1` → 排除測試帳號
- `admin.test_account != ''` → 排除
- `billings.status != 'paid'` → 排除失敗交易
- `billings.email LIKE '%[公司域名]%'` → 標記為內部使用
- `operation_memos.reason = '內部使用'` → 標記或排除

---

## 欄位總覽

| 區塊 | 欄位數 | 說明 |
|---|---|---|
| A. 復刻（自動） | 8 | 品牌, 進案月份, 方案, 版本, 線上線下, 發票狀態, 方案啟用, 方案結束 |
| A. 復刻（部分） | 4 | 產業, 進案金額, 發票月份, 發票金額 |
| A. 復刻（人工） | 5 | KB編號, 電商, 營業項目, 新舊客戶, Sales |
| B1. BQ 擴充 | 15 | workspace_id, email, 統編, 是否公司, 含稅, 稅額, 折扣, 幣別, coupon, 訂單號, 取消原因, 國家, 規模, 用途, 操作人員 |
| B2. Admin 擴充 | 8 | 組織, 讓利%, 服務, feature_flag, 額度刷新, 座位, 測試帳號, 狀態 |
| B3. Amplitude 擴充 | 5 | 活躍用戶x4週, 搜尋x4, 解鎖x4, 收藏x4, 趨勢 |
| **合計** | **45** | 含人工 5 欄（暫不做） |

---

## Part F — 運行模式設計

### F1. 冷啟動回滾（首次執行）

目的：一次性拉取 2024/01/01 起所有交易，建立基線資料。

```
                    冷啟動流程
                    ════════

  ┌─── Step 1: BQ Extract ───────────────────────────┐
  │                                                   │
  │  1a. 線上交易（~290 筆）                            │
  │      SELECT FROM billings                         │
  │      WHERE checkout_at >= '2024-01-01'            │
  │        AND status = 'paid'                        │
  │      JOIN plans, workspaces, workspace_auths,     │
  │           payment_transactions                    │
  │      → raw/online_billings.json                   │
  │                                                   │
  │  1b. 線下交易（~651 筆）                            │
  │      SELECT FROM operation_memos                  │
  │      WHERE created_at >= '2024-01-01'             │
  │        AND reason = '購買方案'                      │
  │        AND operation = '開通購買權限'                │
  │      JOIN admins, workspaces, workspace_auths     │
  │      → raw/offline_memos.json                     │
  │                                                   │
  │  記錄 watermark:                                   │
  │    max_billing_id = MAX(billings.id)              │
  │    max_memo_id = MAX(operation_memos.id)           │
  │    → state/watermark.json                          │
  └───────────────────────────────────────────────────┘
                         │
  ┌─── Step 2: Admin Panel ──────────────────────────┐
  │                                                   │
  │  讀取 admin workspace 資料                          │
  │  （如超過 7 天未更新 → 觸發重爬）                     │
  │  過濾出 Step 1 涉及的 workspace_id                  │
  │  → raw/admin_enrichment.json                      │
  └───────────────────────────────────────────────────┘
                         │
  ┌─── Step 3: Amplitude ────────────────────────────┐
  │                                                   │
  │  API 拉最近 4 週 segmentation                      │
  │  group_by: workspaceId                            │
  │  events: 搜尋KOL, 解鎖KOL, 收藏KOL, Any Active    │
  │  過濾出 Step 1 涉及的 workspace_id                  │
  │  → raw/amplitude_metrics.json                     │
  └───────────────────────────────────────────────────┘
                         │
  ┌─── Step 4: Transform & Merge ────────────────────┐
  │                                                   │
  │  UNION 線上 + 線下                                  │
  │  ├── 方案/版本名稱轉換                               │
  │  ├── 產業 EN→中文 mapping                           │
  │  ├── 線下金額反推未稅                                │
  │  ├── 發票狀態轉換                                   │
  │  ├── 排除 is_test / 內部帳號                        │
  │  LEFT JOIN admin (by workspace_id)                │
  │  LEFT JOIN amplitude (by workspace_id)            │
  │  → output/sales_report.csv                        │
  │  → output/sales_report.json                       │
  └───────────────────────────────────────────────────┘
                         │
  ┌─── Step 5: Load ─────────────────────────────────┐
  │  推到 Notion / 輸出 Google Sheet / 保留 CSV         │
  └───────────────────────────────────────────────────┘
```

**冷啟動產出：**
- `raw/` — 原始拉取資料（可重建）
- `output/sales_report.csv` — 最終報表
- `state/watermark.json` — 記錄進度指標

### F2. 每日增量更新

目的：每天只拉新增的交易，append 到現有報表。

```
                    每日增量流程
                    ════════════

  ┌─── Step 1: 讀取 watermark ───────────────────────┐
  │  讀 state/watermark.json                          │
  │  {                                                │
  │    "last_billing_id": 1060,                       │
  │    "last_memo_id": 5588,                          │
  │    "last_run": "2026-03-11T00:00:00",             │
  │    "total_rows": 941                              │
  │  }                                                │
  └───────────────────────────────────────────────────┘
                         │
  ┌─── Step 2: BQ 增量查詢 ──────────────────────────┐
  │                                                   │
  │  2a. 新線上交易                                     │
  │      SELECT FROM billings                         │
  │      WHERE id > {last_billing_id}                 │
  │        AND status = 'paid'                        │
  │      → new_online[]                               │
  │                                                   │
  │  2b. 新線下交易                                     │
  │      SELECT FROM operation_memos                  │
  │      WHERE id > {last_memo_id}                    │
  │        AND reason = '購買方案'                      │
  │        AND operation = '開通購買權限'                │
  │      → new_offline[]                              │
  │                                                   │
  │  2c. 更新中的欄位（發票狀態/到期日可能變動）            │
  │      SELECT FROM payment_transactions             │
  │      WHERE updated_at > {last_run}                │
  │        AND invoice_status IN                      │
  │            ('is_invoiced','not_invoiced')          │
  │      → updated_invoices[]                         │
  │                                                   │
  │  if new_online + new_offline == 0                 │
  │     AND updated_invoices == 0:                    │
  │     → 跳過，無新資料                                │
  └───────────────────────────────────────────────────┘
                         │
  ┌─── Step 3: Admin 增量 ───────────────────────────┐
  │                                                   │
  │  只查新 workspace_id 的 admin 資料                   │
  │  方式 A: 搜尋 admin 面板                             │
  │  方式 B: 讀現有 json，缺的才補爬                      │
  │  更新 admin 資料快取                                 │
  └───────────────────────────────────────────────────┘
                         │
  ┌─── Step 4: Amplitude 刷新 ───────────────────────┐
  │                                                   │
  │  每次都拉最新 4 週資料（覆蓋式更新）                   │
  │  Amplitude 本身是 rolling window，不需增量            │
  └───────────────────────────────────────────────────┘
                         │
  ┌─── Step 5: Merge ────────────────────────────────┐
  │                                                   │
  │  讀取現有 sales_report.json                        │
  │  ├── APPEND 新交易（去重 by billing_id/memo_id）     │
  │  ├── UPDATE 發票狀態變動的既有列                      │
  │  ├── UPDATE Amplitude 指標（全部覆蓋最新 4 週）       │
  │  ├── UPDATE Admin 資料（有變動的 workspace）          │
  │  寫回 sales_report.csv + .json                    │
  │  更新 watermark.json                               │
  └───────────────────────────────────────────────────┘
                         │
  ┌─── Step 6: Load ─────────────────────────────────┐
  │  推到 Notion / Google Sheet（僅更新差異列）           │
  └───────────────────────────────────────────────────┘
```

### F3. 增量更新的特殊處理

| 場景 | 處理方式 |
|---|---|
| 新的線上交易 | 用 `billings.id > watermark` 抓新紀錄，append |
| 新的線下交易 | 用 `operation_memos.id > watermark` 抓新紀錄，append |
| 發票狀態更新 | 用 `payment_transactions.updated_at > last_run`，UPDATE 既有列的發票欄位 |
| 到期日延長 | 用 `workspace_auths.updated_at > last_run` 且 expire_at 變動，UPDATE 既有列 |
| Amplitude 指標 | 全量覆蓋（rolling 4 週），不做增量 |
| Admin 新 workspace | 新交易涉及的 ws_id 若不在 admin 快取中，觸發單筆爬取 |
| Admin 狀態變動 | 每週一次全量刷新 admin 資料（或手動觸發） |

### F4. 檔案結構

```
sales_report/
├── raw/                          # 原始拉取資料
│   ├── online_billings.json      # BQ 線上交易
│   ├── offline_memos.json        # BQ 線下交易
│   ├── admin_enrichment.json     # Admin 補充
│   └── amplitude_metrics.json    # Amplitude 指標
├── output/                       # 最終報表
│   ├── sales_report.csv          # Excel 可開
│   └── sales_report.json         # 程式可讀
├── state/                        # 狀態管理
│   └── watermark.json            # 增量進度指標
└── config/                       # 設定
    ├── industry_mapping.json     # 產業 EN→中文
    └── plan_mapping.json         # 方案名稱對照
```

### F5. watermark.json 格式

```json
{
  "last_billing_id": 1060,
  "last_memo_id": 5588,
  "last_run": "2026-03-11T00:00:00",
  "last_admin_refresh": "2026-03-10T00:00:00",
  "total_online": 290,
  "total_offline": 651,
  "total_rows": 941
}
```

### F6. 執行方式

```bash
# 冷啟動（首次）
python3 sales_report_pipeline.py --mode=backfill --since=2024-01-01

# 每日增量
python3 sales_report_pipeline.py --mode=incremental

# 強制全量刷新 Admin
python3 sales_report_pipeline.py --mode=incremental --refresh-admin

# 只更新 Amplitude
python3 sales_report_pipeline.py --mode=incremental --amplitude-only
```
