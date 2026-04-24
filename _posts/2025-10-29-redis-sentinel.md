---
layout: post
title: "學習筆記：Redis Sentinel 遷移 — 一個 TPM 跟著 SRE 搞懂高可用的過程"
date: 2025-10-29
categories: [平台架構]
tags: [Software Architecture, DevOps, Data Engineering, Backend]
author: Dennies Hong
---

這篇是我在跟進 KolRadar Redis 架構遷移過程中整理的筆記。整件事從 2023-12 的 POC 到 2024-04 正式上線，都是 SRE/Backend 團隊在執行，我的角色是 TPM，負責協調和追蹤。但因為這個事件牽涉範圍太廣、incident 也踩了不少，我決定好好把這段過程理解清楚，而不只是在 ticket 上打勾。

---

## 為什麼要遷移？起點是一次真實的中斷

當初促成這次遷移的直接原因，是 Crawler 上版時大量 Pod 重啟，K8s Cluster Autoscaler 回收了 Redis 所在的 Node，Redis Standalone 直接中斷。後來問了 SRE 才知道這不是第一次發生，而是 Standalone 架構的先天問題 — Node 被回收就是全線中斷，沒有 failover 能力。

Redis 在 KolRadar 不只是一般意義的 cache，這是我後來才真正理解的部分。它橫跨四個核心服務，用途各不相同：

- **Crawler Service**：Task Queue（List 型別）、Cookie 管理（IG 小帳 cookie、Shadow Cookie）、Job 追蹤（`SPECIFIC:{platform}:running_jobs:*`）。這些資料不是快取，不能遺失。
- **Tiam（Auth Service）**：Session / Auth Token 快取、Snowflake Node ID。Auth Token 可以遺失（重新登入就好），但 snowflake_node_id 不能。
- **Ganesh（AB Testing）**：AB 實驗結果 Bloom Filter、Similar KOL Cache。可以重建，但 Bloom Filter 初始容量設太大（一千萬 = 30 MB/bucket）是後來踩的坑。
- **Datahub API**：Similar KOL Redis Cache、其他 API 快取。

記憶體方面，舊 Standalone `maxmemory` 3750 MB，`maxmemory_policy` 設 `noeviction`（不自動淘汰，滿了直接報錯）。單一爬蟲帳號約 1 KB。

---

## 為什麼選 Sentinel 不選 Cluster？

我一開始以為 Redis Cluster 應該是更完整的方案，後來問了才知道 Cluster Mode 無法滿足需求，原因很具體。

Crawler 使用的 `List` 資料型別無法在 Cluster Mode 下做到分流。Cluster Mode 的 key 會被 hash 到不同的 slot，但爬蟲的 task queue 是一個 List，所有 consumer 都需要操作同一個 List，Cluster Mode 在這個場景下無效。

Sentinel Mode 的優勢：
- 支援故障自動移轉（Failover）至 Replica
- Sentinel 監控 Master 狀態，自動 promote Replica 為新 Master
- `go-redis` 原生支援 Sentinel 連線模式
- 對既有 List 操作完全相容，不需要改 data structure

最終架構是 3 個 Redis nodes（1 Master + 2 Replica）+ 3 個 Sentinel（quorum = 2），部署在 K8s namespace `redis-sentinel`。

<div class="mermaid">
graph TD
    S1["Sentinel 1"]
    S2["Sentinel 2"]
    S3["Sentinel 3"]
    M["Redis Master<br/>node-0"]
    R1["Redis Replica<br/>node-1"]
    R2["Redis Replica<br/>node-2"]

    S1 ---|監控| M
    S2 ---|監控| M
    S3 ---|監控| M
    M -->|同步| R1
    M -->|同步| R2
    S1 ---|quorum=2| S2
    S2 ---|quorum=2| S3

    style M fill:#e05d6f,color:#fff
    style R1 fill:#4a90d9,color:#fff
    style R2 fill:#4a90d9,color:#fff
    style S1 fill:#e8a838,color:#fff
    style S2 fill:#e8a838,color:#fff
    style S3 fill:#e8a838,color:#fff
</div>

---

## POC：核心驗證「Failover 期間 Job 會不會掉」

POC 從 2023-12 啟動，2024-01-26 完成。看了 POC 的 log，才真正搞懂 Sentinel 切換時發生了什麼事。

核心驗證的問題只有一個：在大量併發寫入情況下，執行 Master 節點切換，List 中的 Job 是否會遺失。

實測 log：

```
2024/01/26 16:13:30 data-1606 1607
2024/01/26 16:13:30 data-1607 → (切換發生)
redis: sentinel: new master="mymaster" addr="redis-sentinel-node-2....:6379"
2024/01/26 16:13:32 data-1607 1608  ← 切換後繼續，無遺失
```

看了這段 log 發現，切換期間約 2 秒，List 中的 Job 確實不會遺失。這讓我理解到 Sentinel 的 Failover 機制對這個 queue 場景是可行的。

---

## Go Live：三階段切換策略

<div class="mermaid">
graph LR
    ST1["Stage 1<br/>資料搬移<br/>Cookie/Snowflake ID"]
    ST2["Stage 2<br/>IaC 設定<br/>重啟非 queue 服務"]
    ST3["Stage 3<br/>等 queue 清空<br/>切換 crawler"]
    DONE["上線完成"]

    ST1 --> ST2 --> ST3 --> DONE

    style ST1 fill:#4a90d9,color:#fff
    style ST2 fill:#e8a838,color:#fff
    style ST3 fill:#e05d6f,color:#fff
    style DONE fill:#5bbf72,color:#fff
</div>

2024-04-30 正式切換，隨 Datahub 5.9 Release 一起發佈。問了 SRE 排程的邏輯 — 選時間是有學問的：避開雙週爬蟲排程、避開正常上版時間、選在下午執行，目的是把同時踩雷的機率降到最低。

三階段切換流程：

**Stage 1 — 資料搬移**：Redis Sentinel 建好之後，先手動搬移不可遺失的資料。IG 小帳 Cookie、Shadow Cookie、snowflake_node_id 用 HMSET/SET 手動複製過去。Auth Token 不搬（登出重新登入就好），Crawler Queue 不搬（等舊 queue 消化完）。

**Stage 2 — IaC + 重啟不依賴 queue 的服務**：修改 IaC config，加上 sentinel 設定，重啟不依賴 queue 的 pod（api-service、background、micro、tiam、ganesh）。

**Stage 3 — 等 queue 清空，再切 crawler**：監控舊 Redis queue → 等 queue 清空 → 重啟各平台 crawler（fb/ig/thread/tiktok/tt/yt）→ 驗證新 task 正常消化 → 檢查 auth key 成功寫入 sentinel。

Go 的切換邏輯本身很乾淨：有 sentinel config 就用 sentinel client，沒有就用 redis client。在 `kernel_kit_go` 新增了 Sentinel Redis Provider，應用層幾乎不用改。

---

## Failover 實測：2 秒 vs. 6 秒

上線後做了兩次 Failover 測試，我把結果記下來：

**測試 1**：Kill redis-sentinel-node-0（當時的 Master）→ node-1 被 promote → Failover 時間約 2 秒 → 所有 Pod 自動切換到新 Master。

**測試 2**：Kill redis-sentinel-node-1（剛 promote 的新 Master）→ node-0 恢復後被 promote → Failover 時間約 6 秒。

連續兩次 Failover 均正常，Go redis client + headless service 功能正常。問了 SRE 才知道第二次比較慢是因為 node-0 需要先以 Replica 身份重新加入、同步資料，再被 Sentinel 選為新 Master，步驟比第一次多。

---

## 上線後踩的坑：OOM 和 P1 Incident

遷移本身很順，但上線後兩週就遇到問題。事後回顧這些 incident，才覺得當時低估了記憶體的複雜度。

**2024-05-08 OOM**：`OOM command not allowed when used memory > 'maxmemory'`。46 個 task 中斷，約 92 萬個 kol url 受影響（全部 284 萬）。問了 SRE 才知道根因是 maxmemory 設定不足 — 1.5 Gi 只夠 68% 任務量，估算的時候沒有留夠 buffer。修復方向是調到 2.75 Gi（含 alert 80% buffer），清除廢棄舊資料，Redis 操作加入 retry 機制。

**2024-08-27 P1 Incident**：Redis 記憶體用量達 100%。事後回顧這個 incident，根因是策略爬蟲觸發時遇到 nil pointer panic — 主帳號功能新增後 `platform_user_id` 為 null。觸發頻率是每分鐘跑一次，panic 中斷但部分成功，導致重複 task 不斷寫入 Redis 塞爆記憶體。這個 pattern 值得記住：不是寫的量大，而是因為 crash loop 把同樣的 task 反覆寫入，堆積成 OOM。修復方法：null pointer check。

兩次事件之後，加了兩層防護：
- Grafana Alert：usage > 80% 告警
- 程式 Hard Limit：usage > 95% 限制新增 task（用 `INFO MEMORY` 查詢，O(1)）

---

## Istio Proxy 帶來的奇怪 timeout 行為

Ganesh 掛了 Istio Proxy 之後，Redis 重啟時 dial 會「成功」但拿到無效連線。問了 SRE 才理解問題所在：沒有 Istio 時 3-5 秒就能感知 timeout，有 Istio 時要 150-160 秒才報錯。

Istio Proxy 對 TCP connection 的管理方式不同，讓本來應該快速失敗的連線變成長時間掛在那裡。解法是設定 redis-cluster 的 node 不被 cluster-autoscaler 搬移，避免 Redis 被意外重啟。未來方向是朝 istio-proxy TCP pool 調查。

另外 SRE 還修過 Redis 斷線後 service 超過十分鐘沒重新註冊的 bug，commit 在 `kernel_kit_go` 的 `2fbea269`，是這次遷移附帶發現的問題。

---

## Bloom Filter 容量的教訓

Ganesh 的 AB Testing 用 Bloom Filter，初始容量設一千萬（30 MB/bucket）。問了才知道當初設這麼大是留 margin，但因為 bucket 數量很多，整體吃掉的記憶體遠超預期。後來優化到一百萬（1.8 MB/bucket），大幅降低記憶體佔用。

2025-10 進一步設計了排隊控制方案：
- V1：管制策略爬蟲排隊（只調 cronjob）
- V2：所有任務先進 DB，Queue 有空間再推（較完整，優先選這個）

---

## 回顧整體

Sentinel 遷移本身技術上不難 — Go client 改一行 config、資料手動搬、分批重啟。難的是上線後的 edge case：maxmemory 估算不準導致 OOM、null pointer 導致重複寫入塞爆記憶體、Istio Proxy 讓 timeout 行為完全改變。

事後回顧這整段過程，覺得最有價值的學習不是 Sentinel 的機制本身（那個看文件就懂），而是「Redis 在這個系統裡不只是 cache」這件事 — 一旦把 Redis 當成不可遺失的佇列，記憶體估算、crash loop 保護、連線異常處理，每一個都要認真對待。

---

## 還想更深入的部分

- **Redis 記憶體估算方法論**：這次兩次 OOM 都跟估算不準有關，想搞清楚有沒有更系統化的方式估算各個 data structure 的實際記憶體用量，而不是靠 rule of thumb。
- **V2 方案（任務先進 DB）的架構邊界**：如果 DB 才是 source of truth，Redis 只是緩衝區，那這跟一般的 message queue（Kafka、SQS）有什麼本質差別？什麼情況下值得自己用 DB + Redis 實作 queue，什麼情況下應該直接換成正式的 message broker？
- **Istio TCP pool 和 Redis 的互動**：Istio 讓 dial 成功但連線無效這件事，背後的機制還沒完全搞懂，想找時間把 Istio 的 connection pool 設定讀清楚。
- **Sentinel quorum 的 edge case**：quorum = 2 在 3 個 Sentinel 的情況下，如果同時有 2 個 Sentinel 掛掉，Master 不會被 promote，這個 split-brain 情況下系統的實際行為是什麼？

---

*Dennies Hong — AI/ML Product Manager / 寫技術、寫產品、寫真實經歷。*
