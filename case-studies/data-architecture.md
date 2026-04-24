---
layout: page
title: "Case Study: Data Architecture — 串連八層架構的端到端資料流"
permalink: /case-studies/data-architecture/
---

> From crawling to searching: 475 万 KOL, 80K daily accounts, 15+ ML models, 26 derived features, 7 languages — one pipeline.

## 背景

KOL Radar 是一個幫品牌找網紅合作的 SaaS 平台。平台上每一筆 KOL 資料——粉絲數、互動率、標籤、業配標記、相似網紅——都不是靜態存在的。它們來自 6 個社群平台的爬蟲、經過 ETL 清洗、CDC 即時同步、15 個 ML 模型加工、26 個排程特徵計算，最後透過 GraphQL API 送到用戶面前。

整條管線分屬 6+ 個團隊維護。身為 AI Lab TPM，我的角色是把這些串起來，確保每一層的輸出是下一層預期的輸入。

## 挑戰

- **規模**：475 萬 KOL、每日 80K 帳號爬取、Kafka 週吞吐 500GB、ES 總量 16.9TB
- **多團隊協作**：Crawler Team / DE Team / AI Lab / Backend Team / DS Team，各自有不同的「完成」定義
- **多語系**：7 種語言的搜尋分詞（中/英/日/韓/泰/印尼/越），每種斷詞邏輯完全不同
- **即時性**：品牌方期待「資料是最新的」，但管線從爬蟲到可搜尋涉及 8 個階段

## 八層架構

<div class="mermaid">
graph LR
    A["① 爬蟲<br/>Go · gRPC · Redis FSM<br/>6 平台 · 80K/日"] --> B["② 儲存<br/>GCS + PostgreSQL"]
    B --> C["③ ETL<br/>Argo Workflow<br/>GCS2Flat → Flat2Latest"]
    C --> D["④ CDC<br/>Debezium → Kafka → Flink"]
    D --> E["⑤ ML 加工<br/>15+ 模型 + 26 衍生特徵"]
    D --> F["⑥ ES 索引<br/>Akashic KOL/Post + Hermes"]
    E --> F
    F --> G["⑦ GraphQL<br/>gqlgen + Akashic gRPC"]
    G --> H["⑧ 用戶搜尋<br/>5 種搜尋方式"]

    style A fill:#e67e22,color:#fff
    style B fill:#e67e22,color:#fff
    style C fill:#3498db,color:#fff
    style D fill:#3498db,color:#fff
    style E fill:#9b59b6,color:#fff
    style F fill:#2ecc71,color:#fff
    style G fill:#2ecc71,color:#fff
    style H fill:#1abc9c,color:#fff
</div>

## 關鍵技術決策

| 決策點 | 選擇 | 為什麼 |
|--------|------|--------|
| 爬蟲語言 | **Go** | goroutine 高併發 + 公司標準語言 + 共用 kernel_kit_go |
| 任務佇列 | **Redis Queue + FSM** | 輕量原子操作，不需 RabbitMQ 的複雜 routing |
| 資料同步 | **Debezium CDC** | 零侵入讀 WAL，避免應用層雙寫的一致性風險 |
| 串流引擎 | **Kafka (Strimzi) + Flink** | 高吞吐持久化 + 真正事件驅動（非微批次） |
| 搜尋引擎 | **Elasticsearch** | 一引擎三功能：全文搜尋 + 向量搜尋 + 聚合分析 |
| API 層 | **GraphQL (gqlgen)** | 前端按需取欄位，Field Resolver + DataLoader 解 N+1 |
| ML 策略 | **輕量模型優先 + ONNX** | 自訓 MiniLM 64d > OpenAI embedding（domain 特化） |
| 全球化 | **Country Isolation** | tw + global 雙 index，ES alias 零停機切換 |

## 成果

| 指標 | 數值 |
|------|------|
| KOL 總量 | 475 萬 |
| 每日爬取帳號 | 80,000 |
| ML 模型數 | 15+ |
| 衍生特徵 | 26 個（Crawler/月/週 三頻率） |
| 搜尋語系 | 7 種 |
| CDC 吞吐 | 4,000 rows/sec，lag = 0 |
| Hermes 推理 | 138 ms/request（v1.5） |
| GraphQL 查詢 | 3-69 ms/query |
| Iceberg 成本降 | $2,400 → $466/月（-81%） |

## 我的角色

- **跨團隊串連**：定義每一層的輸入/輸出契約，確保 6+ 團隊的交接點不掉球
- **技術選型參與**：每一層的選型討論都有參與，從 Debezium vs DataStream 到 gqlgen vs graphql-go
- **衍生特徵盤點**：從 Notion 整理出 26 個 DC Features List 的完整清單和更新頻率
- **文件化**：將八層架構整理成可溝通的全景圖，讓新人和非技術利害關係人理解資料流

## 體感

系統的複雜度不在任何單一元件，而在元件之間的銜接。Kafka 很穩、Flink 很快、ES 很強——但 Debezium 的 TOAST value、Flink 的 JVM 交換開銷、ES 的 shard 搬遷卡住，全都發生在銜接處。

每一層的選型看起來都合理，但合理不代表簡單。合理代表「在當時的條件下，這是能力範圍內最務實的選擇」。

---

<a href="{{ site.baseurl }}/資料流全景-八層架構/">→ 完整技術文章：一筆 KOL 資料從被爬到被搜到，中間經過八層架構</a>
