---
layout: post
title: "全球化數據隔離：按國家拆 ES Index 的架構設計"
permalink: /country-isolation/
date: 2026-03-28
categories: [平台架構]
tags: [Software Architecture, DevOps, Data Engineering, Backend]
author: Dennies Hong
---

台灣 KOL 的貼文不能出現在日本的搜尋結果裡。

聽起來理所當然，但要在一個已經運行三年、承載 19 萬 KOL 和上億筆貼文的 ES cluster 上做到，需要一次零停機的架構遷移。這個專案主要由 BE Lead Johnny 設計架構，我負責排程和跨團隊協調。Johnny 後來離職了，我跟他詳細了解過整個設計脈絡，以下是我整理的四個 Phase。

## 為什麼要拆

KOL Radar 最早只有台灣市場。所有 KOL 資料塞在一個 ES index `akashic-kol-v1` 裡，所有貼文按季度切 index `akashic-post-{YYYY}-q{N}`，但不分國家。

市場擴展到日本、新加坡、印尼、澳洲、美國之後，單一 index 的問題浮現。查詢效能下降是其一，更核心的是資源搶佔——台灣是主力市場，每天爬蟲任務量約 80K（IG 40K、Threads 20K、YT 15K、FB 2K），其他國家的大量寫入影響到台灣的查詢延遲，商業影響很大。


技術目標很明確：按國家拆 index，以不影響 KolR 為前提，零停機遷移。

<div class="mermaid">
graph LR
    subgraph 遷移前
        OLD[("akashic-kol-v1<br/>所有國家混合")]
        OLDP[("akashic-post-Q<br/>不分國家")]
    end
    subgraph 遷移後
        TW_K[("akashic-kol-tw")]
        GL_K[("akashic-kol-global")]
        TW_P[("akashic-post-Q-tw")]
        GL_P[("akashic-post-Q-global")]
    end
    ALIAS["akashic-kol<br/>(Alias)"]

    OLD -->|Reindex| TW_K
    OLD -->|Reindex| GL_K
    TW_K --> ALIAS
    GL_K --> ALIAS
    OLDP -->|雙寫入| TW_P
    OLDP -->|雙寫入| GL_P

    style OLD fill:#e05d6f,color:#fff
    style TW_K fill:#4a90d9,color:#fff
    style GL_K fill:#5bbf72,color:#fff
    style TW_P fill:#4a90d9,color:#fff
    style GL_P fill:#5bbf72,color:#fff
    style ALIAS fill:#e8a838,color:#fff
</div>

## Phase 1：KOL Index 隔離

Phase 1 處理 KOL 資料，時間是 2025 年 8 月到 10 月。

在方案評估階段，我們比較了四種方案。差異主要在「向下相容的處理方式」和「開發壓力」上。最終採用方案三，原因是它在三個維度都最優：向下相容可以延後處理、開發壓力最小、風險最小。


方案三的核心思路：search 繼續用舊的 `akashic-kol-v1-*`，upsert-v1 保持不動，新增 upsert-v2 透過 change field 判斷 KOL 的國家，寫入正確的 index（`akashic-kol-tw` 或 `akashic-kol-global`）。切換讀取路徑是後面的事。

上版流程有 10 個步驟，最關鍵的是 Reindex。PRD 環境的 KOL 資料 reindex 會新增 750GB 的磁碟用量，水位預估到 86%。為了壓低 disk 壓力，reindex 期間先把 replica 設為 0。

Reindex query 很直覺——用 `country_code` field 過濾：

```json
// 台灣 index
POST _reindex?wait_for_completion=false
{
  "conflicts": "proceed",
  "source": {
    "index": "akashic-kol-v1",
    "query": {
      "bool": {
        "must": [{ "term": { "country_code": "tw" } }]
      }
    },
    "size": 1000
  },
  "dest": {
    "index": "akashic-kol-tw",
    "version_type": "external_gte"
  }
}
```

但 reindex 不是瞬間完成的。PRD 從 9 月 23 日開始 reindex，9 月 30 日才做 Release 6.16，中間有一週的時間差。這段時間內新寫入的資料不會自動出現在新 index 裡，所以要用 `updated_at >= reindex 開始時間` 做一次時間差補償。

最後是 Alias 切換。把 `akashic-kol` 這個 alias 從舊的 `akashic-kol-v1` 指向新的 `akashic-kol-tw` + `akashic-kol-global`。對上層應用來說，查詢的 alias 名稱不變，但底層已經是按國家隔離的 index 了。

## Phase 2：Post Index 隔離

Post 比 KOL 複雜得多。首先，Post 本身就按季度分 index，所以要拆的不是一個 index，而是每個季度都要拆一次。其次，Post 的資料量遠大於 KOL，不能一次全部 reindex，必須一個 Q 一個 Q 來。


另一個設計上的差異是「雙寫入」。KOL 的方案是先 reindex 再切換，但 Post 因為數量太大、reindex 時間太長，我們選擇在上版後同時寫入舊 index 和新的 tw / global index。這樣即使 reindex 還沒完成，新進來的資料已經在正確的位置了。

雙寫入的判斷邏輯：check alias，如果 `akashic-post-{Q}` 已經指向新 index，就直接寫新 index；否則同時寫舊 index 和新 index。超過三年的 post 不再 upsert，直接忽略。

Post 還有一個特殊情境：KOL 換國家。當一個 KOL 的 country_code 從 TW 改成 JP，他所有的 post 都要搬家——用 `kol_id` 從 ES 查出所有 post，PG 驗證，upsert 到正確國家的 index，再從錯誤的 index 刪除。一年內的 post 強制 sync，兩到三年的 post 走最終一致性。

## Phase 3：Crawler 層隔離

<div class="mermaid">
gantt
    title 四階段隔離時程
    dateFormat YYYY-MM
    section Phase 1
        KOL Index 隔離    :done, p1, 2025-08, 2025-10
    section Phase 2
        Post Index 隔離   :done, p2, 2025-10, 2025-12
    section Phase 3
        Crawler 層隔離    :done, p3, 2025-12, 2026-01
    section Phase 4
        清理與回退       :done, p4, 2026-01, 2026-02
</div>

Crawler 層的隔離不是技術問題，是排程問題。每天 80K 的爬蟲任務如果不分國家，同一個 queue 裡後面的任務等待時間會很長，proxy 資源和 API rate limit 是固定的，cookie 也有限。

我們評估了三種策略：時間分配（12 小時台灣 / 12 小時非台灣）、Queue 細分（按國家開不同 queue）、N 小時啟動（排程分配規則）。最終採用第三種，因為它在 infra 使用分散度和可調整性上都最好。

具體做法：建立分配規則，排出當前時段要爬哪些國家，由排程啟動爬蟲任務。緊急狀況可以暫停策略爬蟲。主力國家的時區都在 +7 到 +9 之間（TW、JP、SG、ID），未來可以用時區切 slice 或按工作日 / 週末分配。

## Phase 4：清理與回退

雙寫入期間會產生髒資料。原本的 cleanup 邏輯只拿一筆資料做判斷，如果 ES 裡有髒資料就可能刪錯。MR #838 修正了這個問題。

回退方案也很重要。如果隔離後出了嚴重問題，我們可以：revert country isolation commit、把 alias 切回舊 index、從 snapshot 恢復。STG 環境實測 120GB 的 index，snapshot 恢復約 10 分鐘。

## Alias 設計策略

整個架構的核心是 ES Alias。KOL 的 alias：`akashic-kol` 同時指向 `akashic-kol-tw` 和 `akashic-kol-global`。Post 的 alias 更複雜：`akashic-post` 指向所有季度的 tw + global index，每個季度也有自己的 alias。

Template 做了繼承設計：base template 定義共用的 mapping 和 settings，global template 繼承 base 但 shard 設為 4 或 6，TW template 繼承 base 但 shard 設為 2。台灣的資料量雖然是主力，但跟全球加起來比，單國的 shard 數不需要那麼多。

## 各層隔離總覽

| 層 | 隔離方式 | 關鍵實現 |
|---|---------|---------|
| ES (KOL) | tw + global 兩個 index | country_code field 分流 |
| ES (Post) | 每 Q 拆 tw + global | Alias 兼容 + 雙寫入 |
| DataCenter | Upsert v2 + Sync v2 | change field 判斷國家路由 |
| Crawler | 時間分配 + 排程策略 | 每 N 小時啟動 + 國家分配規則 |

---

四個 Phase 跨了快半年，兩個 Release（6.16 和 6.17）、三套環境、多個團隊。最難的不是技術，是「什麼時間點切什麼」的排程。reindex 開始時間、release 上版時間、時間差補償時間、alias 切換時間，都要排好，不能影響線上服務。

我一直在想的是：這套架構目前是 tw + global 兩層。如果未來日本市場大到需要獨立 index（不再混在 global 裡），alias 的設計要怎麼擴展？global 要再拆一次嗎？還是一開始就應該每個國家一個 index？這個問題的答案取決於各國市場的成長速度，但架構設計的時間點和市場成長的時間點往往不同步。

---

*Dennies Hong — AI/ML Product Manager / 寫技術、寫產品、寫真實經歷。*
