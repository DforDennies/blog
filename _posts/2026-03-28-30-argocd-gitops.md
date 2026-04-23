---
layout: post
title: "GitOps 實戰：ArgoCD 管理 15+ GKE Cluster 的架構設計"
date: 2026-03-28
categories: [平台架構]
tags: [Software Architecture, DevOps, Data Engineering, Backend]
author: Dennies Hong
---

15 個以上的 GKE cluster，每個都有自己的 namespace 和 helm chart。CDP 有 staging / rc / production 三套，KolRadar 也是三套，AI Datahub 三套，Nexus 三套，再加上一個跑所有基礎設施工具的 CICD cluster。這是 iKala 平台團隊的日常。

在 iKala 擔任 AI Lab TPM，跟 Platform Team 協作超過兩年。這篇不是 ArgoCD 教學，是 cluster 數量超過 15 個之後，GitOps 架構設計怎麼做、踩過哪些坑。

## 一個 CICD Cluster 管全部

先說結論：所有 ArgoCD 的 workload 集中在一個 CICD cluster（GKE `super-resilient-engine`，asia-east1）。這個 cluster 不跑業務服務，只跑基礎設施工具：ArgoCD v3.1.9、Grafana、Loki、Tempo、Mimir、Pyroscope、Traefik、cert-manager、MLflow、Langfuse、Metabase、SonarQube，加起來超過 20 個應用。


從這個 CICD cluster 出發，ArgoCD 透過 GCP Workload Identity 連線到所有遠端 cluster。每個 cluster 的 TLS CA Certificate 存在 `secrets.yaml` 裡，用 SOPS + GCP KMS 加密。GitLab 各 Group 的 Deploy Token 也存在同一份 secrets 裡，ArgoCD 靠這些 credential 去拉各團隊的 Helm repo。

這個設計的好處是「Single Source of Truth」真的只有一個地方：`ikala-developer-platform` 這個 repo。壞處是，如果這個 CICD cluster 掛了，所有環境的部署能力都會暫時中斷。但因為已經部署的服務不受影響（ArgoCD 掛了只是不能同步新版本），所以風險可接受。

## 四種 Application 管理模式

cluster 多了之後，不可能所有 Application 都由 Platform Team 一個人管。我們演化出四種管理模式：


**模式 A：Platform 統一管理。** CICD cluster 上的基礎設施工具，全部定義在 `applications-cicd.yaml`。traefik、grafana、loki、mimir 這些東西由 Platform Team 全權負責，其他團隊不碰。

**模式 B：跨 Cluster ApplicationSet。** 有些元件每個 cluster 都要裝，像 kube-prometheus-stack、traefik、kubecost。用 ApplicationSet 定義一次，自動部署到所有 cluster。實務上這個模式用得不多，大部分已經 comment out 或 disabled。

**模式 C：Team 自管 argocd 目錄。** 這是最靈活的模式。Platform Team 只負責 bootstrap，建好 AppProject 和 cluster credential，然後團隊在自己的 repo 裡維護 `argocd/` 目錄。AI Team 用 `datahub_sys_iac.git/argocd/`，Nexus Team 用 `nexus-iac.git/argocd/`，各自管各自的 Application。

**模式 D：CDP ApplicationSet。** Platform 定義 ApplicationSet template，CDP 團隊透過 `cdp-infra.git` 管理 staging / rc / production 三個環境的差異化設定。

我認為模式 C 是最好的平衡點。Platform Team 不需要理解每個團隊的業務邏輯，團隊也有足夠的自主權。但前提是 AppProject 的 sourceRepos 和 destinations 要設對，不然團隊可能不小心部署到錯誤的 cluster。

## App of Apps 自管理

ArgoCD 本身的設定也用 ArgoCD 管理，這聽起來像套娃，但確實是最穩的做法。頂層有一個 `argocd-apps` Application，管理所有 AppProject 和頂層 Application 的定義。底下再分：

- `argocd-apps-cicd`：CICD cluster 的基礎設施
- `argocd-apps-shared`：跨 cluster 的共用元件
- `argocd-apps-ai`：AI Team 自管
- `argocd-apps-nexus`：Nexus Team 自管
- `argocd-apps-cdp-infra-{staging,rc,production}`：CDP 三環境

整個結構是一棵樹。改任何一個 Application 的設定，都是提 MR 到 `ikala-developer-platform`，review 之後 ArgoCD auto sync。

## 踩過的坑


**ArgoCD 升級 2.14 到 3.0。** 2026 年初做了一次大版本升級，原因是 v2.14 內嵌的 K8s OpenAPI schema 太舊，無法識別某些新的 status field。升級過程遇到三個問題：

1. Redis RDB 格式不相容。ArgoCD 內建的 Redis 從 v7.4.2 降到 v7.2.8，RDB 檔案讀不了。解法是刪除 RDB 檔案重啟。
2. `secrets.yaml.dec` 空檔案。helm-secrets plugin 在某些情況下會產生空的 `.dec` 檔案，然後跳過解密。解法是手動刪除 `.dec` 重新執行。
3. Resource Tracking 模式變更。v3 預設改用 annotation-based tracking，但有 bug。我們改回 `label` tracking 才穩定。

**Traefik Plugin 外部依賴。** Traefik 啟動時會從外部 registry 下載 Plugin，如果下載失敗，所有 Middleware 直接停用。這意味著一個第三方服務的不穩定，可能讓整個 ingress 層失效。我們的解法是改用 Local Plugin 模式：initContainer 從 GitHub 下載 zip，解壓到 `/plugins-local/src/`，Traefik 從本地載入。不再依賴外部 registry。

**GCP Credentials 的隱性陷阱。** 如果在 Pod 裡設了 `GOOGLE_APPLICATION_CREDENTIALS` 環境變數，即使值是空的或指向不存在的檔案，Workload Identity 也會失效。因為 GCP SDK 會優先讀這個環境變數，找不到 credential 就直接報錯，不會 fallback 到 Workload Identity。解法很簡單：刪掉這個環境變數。

**KolRadar 的歷史債務。** KolRadar 的 ApplicationSet 是過去手動 `kubectl apply` 上去的，不在 `ikala-developer-platform` 的版本控制中。這代表如果 ArgoCD 需要重建，KolRadar 的設定無法從 repo 還原。我們建議他們採用模式 C 納管，但截至目前還沒完成。

## 加入新 Cluster 的流程


整個流程三步：

1. 在 `secrets.yaml` 加入新 cluster 的 credential（server URL、CA certificate、execProviderConfig），然後 SOPS 重新加密。
2. 在 `values.yaml` 建立 AppProject，設定 sourceRepos、destinations、roles。
3. 提 MR，Platform Team review，merge 後 ArgoCD auto sync。

如果是全新的團隊，額外需要在 `credentialTemplates` 加入 GitLab Deploy Token。目前已有 7 組 credential template，涵蓋 [6 組內部團隊]。

## 成本意識

一個常被忽略的點：基礎設施的成本控制。我們在 ES 的 disk type 上做了調整，hot tier 從 pd-ssd 換成 pd-balanced，光這一項就省了 $84/月。warm-cold tier 同樣的操作再省 $49/月。Dev 和 Staging 環境盡量用 Spot 實例。每次做 disk scale 都有成本計算記錄。

這些數字看起來不大，但乘以 15+ 個 cluster 和 12 個月，累積起來的金額就值得認真對待。

---

管 15 個 cluster 跟管 3 個最大的差異不是技術複雜度，是「誰負責什麼」的邊界設計。四種管理模式就是在回答這個問題。

我一直在想的是：當 cluster 數量繼續增長，模式 C 的「Team 自管」會不會變成另一種碎片化？每個團隊的 Helm chart 風格不同、命名規範不同、resource limit 設定不同。統一規範和團隊自主之間的平衡點在哪裡，還沒有答案。

---

*Dennies Hong — AI/ML Product Manager / 寫技術、寫產品、寫真實經歷。*
