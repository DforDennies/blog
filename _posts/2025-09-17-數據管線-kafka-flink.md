---
layout: post
title: "學習筆記：KolRadar 數據管線架構——Kafka、Flink 到 Iceberg"
permalink: /data-pipeline-kafka-flink/
date: 2025-09-17
categories: [平台架構]
tags: [Software Architecture, DevOps, Data Engineering, Backend]
author: Dennies Hong
---

這份筆記是我在理解 DE Team 建的 CDC 數據管線時整理的。我是 TPM，不是 DE，所以有很多地方是跟 DE 討論後才搞清楚的。整個系統比我最初想的複雜很多，光是 Kafka 就有好幾條不同定位的管線在跑。

---

## 管線全景

一開始我以為就是一條 pipeline，跟 DE 討論後才發現整個數據管線群分成五種不同的管線，各自有不同的定位：

```
PostgreSQL → Debezium CDC → Kafka → Flink → ES / Iceberg
```

這條是核心。但實際上同時在跑的包含：

1. **CDC Data Pipeline**：Debezium + Kafka + Flink，PG → Kafka → ES / Iceberg，即時同步
2. **EventHub**：Kafka 為底，AI Micro Service 的事件分發
3. **PubSub Pipeline**：GCP Pub/Sub，CDP 事件接收和爬蟲觸發
4. **Backup Pipeline**：Argo Workflows + Flink，ES/Scylla → GCS → BQ
5. **Browse History**：Kafka + Flink，Chrome Extension 瀏覽紀錄 → ES

<div class="mermaid">
graph LR
    PG[("PostgreSQL<br/>WAL")]
    DEB["Debezium CDC"]
    KFK["Kafka<br/>(Strimzi on K8s)"]
    FLK["Flink"]
    ES[("Elasticsearch")]
    ICE[("Iceberg<br/>Data Lake")]
    EH["EventHub"]
    AI["AI Micro Services"]
    PS["GCP Pub/Sub"]
    CDP["CDP / Crawler"]
    BH["Browse History"]
    CHR["Chrome Ext"]

    PG -->|WAL| DEB --> KFK
    KFK --> FLK
    FLK --> ES
    FLK --> ICE
    KFK --> EH --> AI
    PS --> CDP
    CHR --> KFK --> BH --> ES

    style PG fill:#4a90d9,color:#fff
    style KFK fill:#e8a838,color:#fff
    style FLK fill:#e05d6f,color:#fff
    style ES fill:#5bbf72,color:#fff
    style ICE fill:#7b68ee,color:#fff
    style DEB fill:#6c8ebf,color:#fff
</div>

跟 DE 討論後理解到，Kafka 和 PubSub 雖然功能上看起來重疊，但定位不同：Kafka 負責高吞吐量持久化串流（CDC 管線、EventHub），自建 Strimzi on K8s，由 DE Team 管理；PubSub 負責事件驅動場景（CDP、爬蟲觸發、Webhook），GCP 託管，Backend Team 管理。兩者分工清楚，不是重複。

---

## Debezium CDC：從 WAL 到 Kafka

這邊問了才知道，Debezium 選型的原因很直接：當時 Google DataStream 對 PostgreSQL 的支援只有孵化階段，Debezium 比較成熟。部署類型選 Kafka Connector，排除了 Debezium Server（孵化階段）和 Embedded（客製化 effort 過重）。

PG 端要設定 `wal_level=logical`，建立具備 REPLICATION 權限的帳號，所有被監聽的 Table 需有 Primary Key。Publication 手動建立，逐一 ADD TABLE。

Debezium Connector 監聽的表用 REGEX 定義，涵蓋所有平台的 profile、post、comment、hashtag、mention，以及 ML 產出的 autotag、sentiment、language detection 等。

看了 CDC 的 config 才發現一個我之前完全不知道的坑：**TOAST Values**。PostgreSQL 的 `pgoutput` plugin 在 UPDATE 時，若未變更的 TOAST 欄位不會包含在 WAL 中，Debezium 收到的值會是 `__debezium_unavailable_value`。DE 的解法是 `ALTER TABLE REPLICA IDENTITY FULL`，會增加些許 CPU 但確保資料完整性。這個問題如果沒有踩過根本不會想到。

另外，Partition Table 用 `ByLogicalTableRouter` 做 Reroute，把 `_202401`、`_202402` 這種分區表合併到同一個 Kafka Topic。這個設計邏輯我覺得很乾淨。

---

## Kafka 集群規格

跟 DE 討論後理解到，他們用 Strimzi 部署在 K8s 上，目前是 Kafka + ZooKeeper 模式（還沒遷移至 KRaft）。

PRD 環境是 3 個 Kafka Nodes，每個 2 CPU / 4G / 600G，總 Disk 1.8TB。容量計算方式是：Replica Set = 3，每月增加 100GB，Retention 1 個月約 200GB × 3 = 600GB/node。

高流量 Topic（IG Post、Profile、Comment）設 18 個 Partition，中流量 Topic 設 9 個。Topic cleanup policy 用 compact，壓縮用 lz4。Retention 設 7 天 Delete Policy — 這邊問了才知道，7 天的設定是有意識的，如果管線有問題，理論上應該在 7 天內發現並解決。

有個真實踩過的坑：US 百網紅匯入後遇到 Message Too Large 問題，Binary Column（thumbnail）過大，單筆可達 9.6MB。後來把 `producer.max.request.size` 和 `message.max.bytes` 都調到 30MB，Flink 端先跳過過大的 message。

跟 DE 確認過的數字：每 10 秒從 Kafka 寫入下游約 4 萬筆，大約 4,000 rows/sec。Consumer message behind 穩定在 0，Auto Scaler 有效。

---

## Flink CDC Streaming

跟 DE 討論後理解到，Flink 的技術演進經歷三個階段，每個階段都是為了解決前一個階段的問題：

<div class="mermaid">
timeline
    title Flink CDC 技術演進
    2023 : PyFlink + ScyllaDB Sink
         : JDBC 版本過舊
         : MapFunction 自行實作
    2024 : 遷移至 Java Flink
         : Sink 改為 Elasticsearch
         : 效能大幅提升
    2025-2026 : Java Flink CDC 到 ES
              : 同步測試 Flink CDC 到 Iceberg
              : 邁向 Data Lakehouse
</div>

1. **初期（2023）**：PyFlink 開發，ScyllaDB 作為 Sink。ScyllaDB JDBC 版本過舊，只能用 MapFunction 自行實作 Insert
2. **中期（2024）**：遷移至 Java Flink，Sink 改為 Elasticsearch
3. **現行（2025-2026）**：Java Flink CDC → ES，同時測試 Flink CDC → Iceberg

PRD 環境用 1 個 JobManager（2 CPU / 2G）+ 3 個 TaskManager（2-4 CPU / 6G），每個 TM 2 個 Slot。

Window 策略也有演進。看了設計文件才發現，最初用 Non-Keyed Window + Timer Trigger（3-5 秒），但 Python 和 JVM 間的資料交換有額外成本，Timer Trigger 不穩定。改用 Count Trigger + 長間隔 Timer 後，引入 Checkpoint（每分鐘，輕量）和 Savepoint（每小時，Heavy）確保資料不遺漏。

驗證結果：PRD IG Post 約 1,600 萬筆資料成功匯入，AutoScaler 資源提升如預期，縮容回一台 TaskManager 也順利完成。

---

## Flink CDC → Iceberg

這邊問了才知道，Flink CDC → Iceberg 是通往 Data Lakehouse 的關鍵路徑。用 Flink SQL 定義 Kafka Source（Debezium JSON format）和 Iceberg Sink（Nessie Catalog + GCS），Checkpoint interval 設 10 秒。

CDC 持續寫入會產生大量小檔案，這是我之前沒想到的問題。跟 DE 討論後理解到他們的解法：由 Argo CronJob（不是 Flink）定時執行 `optimize`，不佔用 Flink 資源。這個設計選擇我覺得很有意思，讓 Flink 專注做串流，合併檔案交給排程任務，職責分離。

---

## ScyllaDB：上場又退場

看了架構演進記錄才發現，ScyllaDB 曾作為 CDC Pipeline 的中間儲存層，選用原因是高寫入吞吐量和寬列存儲。PRD 環境開了 3 個 Nodes，每個 4 CPU / 32G / 5.5TB，總計 16.5TB，規模不小。

但 2025 年下半年執行了退役，資料遷移至 ES。跟 DE 討論後理解到，退役的核心原因是中間層增加了系統複雜度，而 ES 已經能直接作為 CDC 的 Sink，省掉一層轉換。現在 BQ backup schema `renata_rawdata_all_datalake` 保留了原 Scylla 的資料。這種「加進來再拿掉」的演進在複雜系統裡其實很常見。

---

## 2025 H2 Kafka 遷移規劃

這邊問了才知道，DE 有計劃把 Kafka 升版，目標版本是 Strimzi Operators 0.47、Kafka 4.0.0。如果 KRaft 不穩定就維持 ZooKeeper。需要移轉的元件包含 Flink（Data Lake）、Debezium、AI EventHub（akashic、RSS、GA Events）以及 40925。

新增 Table 到 CDC 管線的 SOP 也已經標準化，跟 DE 確認過的步驟是：加入 Publication → 設定 REPLICA IDENTITY → 確認 Kafka Topic 自動建立 → Flink 程式新增對應 mapping → 觸發初始資料同步。有個細節是因為 `snapshot.mode=never`，初始同步需要手動 UPDATE 一下 `updated_at` 欄位觸發 CDC，這個如果不知道會卡很久。

---

## 還想更深入的部分

- **跨表業務邏輯**：CDC 管線目前是以 Table 為單位訂閱，但有些跨表的業務邏輯（比如 KOL 的 profile 更新要連帶影響 post 的某些欄位），在 CDC 層面要怎麼處理？目前是靠下游 ETL 補，但這樣時效性就打折了，想更了解這塊的設計取捨
- **KRaft 遷移細節**：Kafka 4.0.0 + KRaft 模式的遷移計劃，想了解 ZooKeeper → KRaft 的風險評估和 rollback 策略
- **Iceberg compaction 策略**：Argo CronJob 執行 `optimize` 的頻率和觸發條件是怎麼設計的，小檔案問題的量級大概是多少
- **Flink Savepoint 管理**：每小時 Savepoint 的保留策略和 recovery 流程，實際上有用到過嗎

---

*Dennies Hong — AI/ML Product Manager / 寫技術、寫產品、寫真實經歷。*
