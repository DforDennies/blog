---
layout: post
title: "Kolr Palette：獨立 AI Agent 找網紅系統的架構設計"
permalink: /kolr-palette/
date: 2026-01-28
categories: [AI 產品策略]
tags: [AI, Product Management, SaaS, Product Strategy]
author: Dennies Hong
---

Kolr Palette 的內部代號是 40925。從單代理 MVP 長成完整 Agent 平台，花了不到一年。

2025 年 4 月第一版上線，到 2026 年初已經跑過九個版本、用過四種 LLM Supervisor 模型、整合了 MCP 協議、做出了能自動生成 PowerPoint 報告的 PPT Agent。這篇拆解 40925 的架構演進、技術選型，和我們在過程中踩過的坑。

---

## 為什麼要做獨立 Agent

KOL Radar 本體是搜尋工具——用戶設篩選條件、看結果、解鎖 KOL 資料。2023 年 AI Search 上線後，用自然語言取代了部分篩選操作，但本質上還是「搜尋 → 結果列表 → 手動挑選」。

40925 要解決的問題是：**能不能讓 AI 直接幫用戶完成整個分析工作流程？**

不是搜尋完給你一個列表讓你自己看，而是：
- 多輪對話釐清需求
- 自動搜尋、篩選、比較
- 生成分析報告（PPT / PDF）
- 記住你的偏好，下次更精準

最終目標是一個坐在 KOL Radar 資料之上的多代理對話層，自動化分析師的工作流程。

---

## 五個 Phase 的架構演進

### Phase 1：架構設計（2025-01 ~ 2025-03）

先花了兩個月探索框架。評估了 Dify 和 Oneness-AI，最後決定自建 Multi-Agent Core。


<div class="mermaid">
graph TD
    P1["Phase 1<br/>架構設計<br/>2025-01~03"]:::p1 --> P2["Phase 2<br/>多代理擴展<br/>2025-04~06"]:::p2
    P2 --> P3["Phase 3<br/>功能深化<br/>2025-07~09"]:::p3
    P3 --> P4["Phase 4<br/>記憶與協議遷移<br/>2025-10~12"]:::p4
    P4 --> P5["Phase 5<br/>模型現代化<br/>2026-01+"]:::p5
    P2 -.- F2["PPT Agent v1<br/>4種協調模式"]:::feat
    P3 -.- F3["Artifacts<br/>Deep Research<br/>Dynamic MCP"]:::feat
    P4 -.- F4["Mem0 記憶<br/>MCP → Streamable HTTP"]:::feat
    style P1 fill:#e74c3c,stroke:#c0392b,color:#fff
    style P2 fill:#f39c12,stroke:#e67e22,color:#fff
    style P3 fill:#3498db,stroke:#2980b9,color:#fff
    style P4 fill:#9b59b6,stroke:#8e44ad,color:#fff
    style P5 fill:#2ecc71,stroke:#27ae60,color:#fff
    style F2 fill:#fdebd0,stroke:#f39c12
    style F3 fill:#d6eaf8,stroke:#3498db
    style F4 fill:#e8daef,stroke:#9b59b6
</div>

不直接用開源框架，主要是兩個原因：
1. 我們需要深度整合 KOL Radar 的資料查詢（走 MCP 協議），現有框架的工具整合不夠靈活
2. Agent 的對話流程和 session 管理需要跟我們自己的用戶系統打通

同期部署了 LangFuse 作為觀測平台。這個決策後來證明非常關鍵——沒有 tracing，多代理系統的 debug 幾乎不可能。

### Phase 2：多代理擴展（2025-04 ~ 2025-06）

MVP 上線後，迅速從單代理擴展到四種 orchestration 模式：

- **Swarm**：群體協作，多個 agent 並行處理不同子任務
- **Selector-Group**：路由選擇，根據意圖分類把請求送到對應的專家 agent
- **Round-Robin**：輪詢分配，負載均衡
- **Supervisor Model**：一個監督者 agent 調度其他 worker agents

同時上線了 PPT Agent v1——第一個能產出交付物的 Agent。從自然語言查詢到 PowerPoint 輸出，整個流程自動化。

這個階段也建了 Experiment Manager v1.0，用 LLM-as-a-Judge 做系統性的品質評估。不然只靠人工看對話紀錄，根本跟不上迭代速度。

### Phase 3：功能深化（2025-07 ~ 2025-09）

三個重點：Artifacts、Deep Research、Dynamic MCP。

**Artifacts** 用 KV-Store + RDB 做持久化輸出。Agent 生成的內容不再只存在對話記錄裡，而是可以跨 session 保存和引用。

**Deep Research Phase 2** 做了 Multi-hop 研究管線：計劃 → 搜尋 → 分析 → 總結。加上 SequentialThinking（結構化多步推理），讓 Agent 可以處理複雜的研究問題。

**Dynamic MCP tool registration** 讓 worker agents 在運行時動態註冊工具。這解決了一個實際問題：不同類型的任務需要不同的工具組合，寫死的工具列表太不靈活。

### Phase 4：記憶與協議遷移（2025-10 ~ 2025-12）

Agent 沒有記憶是最被用戶抱怨的問題。Phase 4 做了兩輪 Memory POC：

第一輪評估了 Mem0 和 a-mem。第二輪把 Mem0 + a-mem + Autogen query layer 整合在一起，實現跨 session 的持久記憶。

記憶系統的底層用 Elasticsearch 做向量搜尋（1,536 維 embedding），AWS Neptune 做知識圖譜，Azure OpenAI gpt-4.1-mini 做語意處理。記憶以 user_id / agent_id / run_id 三個維度索引。

同期做了 MCP 協議遷移：從 SSE 升級到 Streamable HTTP。Supervisor 模型也從 GPT-4.1 換成 Claude Sonnet 4.5。


### Phase 5：模型現代化（2026-01+）

嘗試 Haiku 4.5 作為 Supervisor（降低幻覺，使用 todos 機制），也探索了 Grok。個人化回應風格按網紅類型調整。

---

## 技術棧全覽

| 層級 | 技術 |
|------|------|
| LLM Provider | Azure OpenAI (GPT-4.1, o3-mini)、Anthropic (Claude Sonnet 4.5, Haiku 4.5) |
| Agent Framework | 自建 Multi-Agent Core |
| Observability | LangFuse |
| Memory | Mem0 + a-mem + Autogen query layer |
| Tool Protocol | MCP (Streamable HTTP) |
| Web Search | Serper + Perplexica (LangChain + SearxNG) |
| Frontend | Shadcn UI |
| Payment | Stripe |
| Evaluation | Experiment Manager + LLM-as-a-Judge + Dueling Bandits |
| Database | PostgreSQL (Goose migration, `agentic` schema) |
| Admin | Next.js 15 + React 19 (palette-admin.kolr.ai) |

後端用 Go，服務間走 gRPC / Protobuf。Protobuf 定義了三個核心服務：
- **AgentService**：`RunStream(Task) -> stream StreamOutput`，支援 text / ChatMessage / ChatMessageList 三種輸入
- **MCPCacheService**：快取 MCP 工具呼叫的 request / response
- **ArtifactService**：管理生成物的下載 URL

---

## 評估系統：怎麼知道 Agent 變好了

這是我認為 40925 做得最紮實的部分。

<div class="mermaid">
graph LR
    INPUT["Agent 輸出"]:::input --> JUDGE["LLM-as-a-Judge"]:::judge
    INPUT --> DUEL["Dueling Bandits<br/>Thompson Sampling"]:::duel
    JUDGE --> SCORE["品質評分"]:::score
    DUEL --> RANK["模型排名"]:::rank
    SCORE --> REPORT["Experiment Report<br/>品質/延遲/工具使用"]:::report
    RANK --> REPORT
    style INPUT fill:#f39c12,stroke:#e67e22,color:#fff
    style JUDGE fill:#e74c3c,stroke:#c0392b,color:#fff
    style DUEL fill:#3498db,stroke:#2980b9,color:#fff
    style SCORE fill:#9b59b6,stroke:#8e44ad,color:#fff
    style RANK fill:#1abc9c,stroke:#16a085,color:#fff
    style REPORT fill:#2ecc71,stroke:#27ae60,color:#fff
</div>

**LLM-as-a-Judge** 用兩種模式：Direct Evaluation（給分）和 Pairwise Comparison（比較兩組輸出）。分數轉換支援 identity、linear、binary 三種方式。


**Dueling Bandits** 是更進階的模型比較機制。透過人類標記者做 pairwise comparison，用 Thompson Sampling 自動平衡 exploration 和 exploitation：

- Phase 1（Bootstrap，模型 < 5 comparisons）：優先標記數據最少的 pair
- Phase 2（Thompson Sampling）：每個模型從 Beta(wins+1, losses+1) 取樣勝率，選 combined score 最高的 pair

統計顯著性用 confidence interval 判定：model A 的 CI 下界 > model B 的 CI 上界，就判定 A 顯著優於 B。50% 隨機 swap 防位置偏差。

**Experiment Manager CLI** 提供 `run`（平行執行多組 agent 配置）、`judge`（批次 LLM 評分）、`report`（quality / latency / tool_usage 維度分析）三個指令。

---

## 真實踩到的坑

從 1,189 個 ticket 裡可以看到最常見的痛點：

1. **Report 生成時間長**：至少等 10 分鐘，思考時間 ≥ 2 分鐘。用戶體驗很差，但這是 multi-hop 研究管線的代價
2. **模型記憶問題**：Agent 有機率忘記之前的對話。Memory 系統上線後改善了，但 context window 限制還是會造成資訊遺失
3. **MCP 連線穩定性**：多次 Bad Request、Timeout。從 SSE 遷移到 Streamable HTTP 後穩定很多
4. **Token 超標**：Context Window 延伸問題、4.x MB 檔案上傳導致 OOM
5. **多語系一致性**：用戶用中文問，Agent 有時候用英文回。這是 prompt 層的問題
6. **時間認知錯亂**：模型的訓練資料和實際時間衝突，問「最近」的數據會錯

---

## 商業化

2025 年 12 月開始大量商業化的 ticket：Stripe 金流串接、訂閱方案、Checkout Session API、用戶 Credit 系統、推薦碼機制。

Admin 後台（palette-admin.kolr.ai）用 Next.js 15 + React 19 + shadcn/ui 建，管理用戶、方案、Credit 紀錄、Feature Card。

產品對外 branding 叫 KolrChat，但內部一直用 40925 這個代號——內部記錄提到「40925 代號很好，引起大家好奇興趣」。

---

## 客戶驗證

40925 的 Agent 基礎設施被用在兩個外部客戶專案：Client B（工業文件解析）和Client C（健康管理）（LINE 聊天機器人）。這兩個案子同時驗證了 Agent 架構在不同場景的適用性。

---

40925 從 MVP 到九個版本，最大的學習是：Agent 系統的難度不在「讓 LLM 生成回答」，而在工程化——記憶持久化、工具動態註冊、多代理協調、品質評估、觀測追蹤。每一個都是獨立的工程問題，組合在一起的複雜度是乘法不是加法。

我還在想的一件事是 Supervisor 模型的選擇。從 GPT-4.1 換到 Claude Sonnet 4.5 再試 Haiku 4.5，每次切換都帶來不同的 trade-off（推論速度 vs 品質 vs 幻覺率 vs 成本）。有沒有可能根據任務類型動態選擇 Supervisor？這個方向我們還沒認真試過。

---

*Dennies Hong — AI/ML Product Manager / 寫技術、寫產品、寫真實經歷。*
