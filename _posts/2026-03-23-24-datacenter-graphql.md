---
layout: post
title: "統一數據查詢入口：GraphQL + gRPC 封裝 ES"
date: 2026-03-23
categories: [平台架構]
tags: [Software Architecture, DevOps, Data Engineering, Backend]
author: Dennies Hong
---

前端要 KOL 資料打一個 API，後端要打三個不同的 ES index。這中間的翻譯層，就是 DataCenter。

產品功能圍繞搜尋：搜 KOL、搜貼文、分析表現、推薦相似網紅。底層全打 Elasticsearch，但 ES 查詢語法複雜、index 分散（KOL index、Post index 按季度分、Text Action index 各自獨立），前端直接對接 ES 的話，維護成本會爆炸。

DataCenter 的設計目標很明確：前端只看到 GraphQL API，後端只看到 gRPC protobuf。中間的 ES 操作、AB Testing 分流、權限驗證、N+1 問題，全部在這一層解決。

## 架構分層

整個 DataCenter 分三層：

```
Frontend (Gaia SDK) → GraphQL over HTTP
    ↓
Datahub API (Interface) → Tiam (Auth) / Ganesh (AB Testing)
    ↓ gRPC
Akashic (MicroService) → Elasticsearch
```

**Datahub API** 是 GraphQL 入口，port :10102，負責接收請求、驗證 auth（透過 Tiam 用 app-id + app-secret）、判斷 AB Testing（透過 Ganesh），然後轉發給 Akashic。

**Akashic** 是核心 — 封裝所有 ES 的讀寫操作，提供 gRPC 介面。它是 ES 的唯一寫入入口（Crawler/ETL → Akashic → ES），也是主要的查詢入口。Protobuf 定義在 `kernel_kit_go` 中管理。

**Gaia SDK** 是前端封裝層，把原本直接使用 ES client 的程式碼轉換為透過 GraphQL API 存取。封裝的戰略價值在於：所有 ES 操作集中到中央位置，未來替換 ES 只需改 SDK 不必改全部 codebase。

一個 KOL Search 的請求流程：

```
Request → Tiam(Auth?) → Ganesh(AB?) → Akashic(1st ES Search)
  → Resolve fields
    → similar_kol (field resolver)
    → related_kol (field resolver)
    → PostConnections? → Akashic(2nd ES Search)
  → Response
```

## GraphQL Schema 設計


Go 語言，用 gqlgen v0.17.44 做 code-first 的 GraphQL 框架。Schema 按功能拆分成 `common.graphqls`、`kol.graphqls`、`post.graphqls`、`analyze.graphqls`，Release 5.5 後重構的結果。

主要 Query 包含 `kolSearch`、`postSearch`、`multiplePostSearch`、`analyze`、`quickSearch`、`kolAnalyze`。Pagination 用 Cursor-based Connection pattern，支援 ES 的 search_after，避免 deep pagination 問題。

Cursor 實作是 Base64 編碼的 JSON 結構，包含 sort 資訊和 search_after 值。default limit 10，max limit 100。Analyze KOL 支援 bidirectional cursor（after + before 擇一，不可同時帶入）。

Reduce I/O 策略是從 request/response field 決定要做的事情。如果 kolSearch response 沒用到 similar_kol / related_kol，就不去 DB 拿 record。如果 postSearch 沒有 customized_tag_list，就不拿 mapping。

## N+1 問題：從暴力查詢到 DataLoader


KOL Search 回傳 N 個 KOL 時，原本每個 KOL 都會各自查詢 `similar_kol` 和 `related_kol`，典型的 N+1 問題。`customized_tag` 的 mapping 查詢也一樣 — MultiplePostSearch 會呼叫 3 次 `TiamUserService/CustomizedTagMappingGet`，KolSearch 的 PostConnection 更糟。

解法分兩步走：

**Step 1（Release 5.6+）**：把 `similar_kol` 和 `related_kol` 從 preload 改為 field resolver。gqlgen 會在填入 response 時才 call 對應的 field resolver function，透過 YAML 配置 `bind to method name`。`ab_testing_id` 帶入 operation context 的 variables，`kol_id` 從 field context 取得。

**Step 2（2024-Q4）**：引入 DataLoader 做批次化。評估了四個套件後選用 `vikstrous/dataloadgen`（gqlgen 官方推薦，綜合評分 91/100）。初版用的是 `graph-gophers/dataloader`（來自官方 example），但高負載時記憶體會爆炸，POC 後遷移。

customized_tag N+1 的解法不同 — 把 customized_tag 邏輯移到 akashic usecase 統一處理，只查詢一次 mapping。

## ES Index 與 Country Isolation


Akashic 管理的 ES Index 分三類：

- **KOL Index**：`akashic-kol`（alias）→ `akashic-kol-tw`（台灣）+ `akashic-kol-global`（全球排除台灣）
- **Post Index**：`akashic-post-{year}-q{quarter}`，按季度分 index。Country Isolation 後再拆成 `-tw` 和 `-global`
- **Text Action Index**：獨立 index，用於推薦文案行為

Country Isolation 的 KOL 拆分（Release 6.16, 2025-Q3）是個不小的工程：

1. Reindex `akashic-kol-v1` → `akashic-kol-global` + `akashic-kol-tw`
2. 上版（雙寫邏輯 — 同時寫入新舊 index）
3. 設定 Alias，查詢時透過 alias 同時搜尋兩個 index
4. 補 reindex 時間差資料
5. 移除舊寫入邏輯

Upsert 邏輯的改變是關鍵：Upsert 前先刪除該 KOL（避免跨 index 重複），依 country_code 路由到對應 index。寫入 tw 的寫 `akashic-kol-tw`，其他的寫 `akashic-kol-global`。

Post 的 Country Isolation（Release 6.17, 2025-Q4）更複雜，因為 post 按季度分 index，需要逐季 reindex。用「偷時間策略」— 提前 2 週開始 reindex，上版日只需做補差量。STG 實測 120GB index restore 約 10 分鐘。

Plan B（回退方案）很簡單：退版 + Alias 調整回原始 index。

## Analyze KOL 的複雜 Aggregation

Analyze KOL 用到 ES 的 terms aggregation + nested aggregation，group by kol_id 後算 comment_count、engagement_count、engagement_rate、follower_count 的 stats，加上 date_histogram 做趨勢分析、top_hits 取最新和最高互動的貼文。

這是整個 DataCenter 裡最重的查詢。Pagination 也特別設計了 bidirectional cursor 支援前後翻頁。

---

DataCenter 把複雜度封裝在中間層。前端不需要知道 ES 的 query DSL、不需要知道 index 按季度分、不需要知道 Country Isolation 改了 index 結構。後端要換資料來源，只需要改 Akashic 這一層。

但我還在想的是：Akashic 現在是 ES 的唯一入口，這個「唯一」帶來了封裝的好處，也帶來了瓶頸的風險。當 Analyze KOL 的複雜 aggregation 把 ES 吃滿的時候，是該在 Akashic 層做快取，還是該把部分查詢分流到其他資料來源（比如 Trino/StarRocks）？

---

*Dennies Hong — AI/ML Product Manager / 寫技術、寫產品、寫真實經歷。*
