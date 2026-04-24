---
layout: page
title: "Case Study: KOL Palette — AI Agent 產品從零到一"
permalink: /case-studies/kolr-palette/
---

## 背景：為什麼要在搜尋工具之上建獨立 Agent

KOL Radar 本體是搜尋工具。用戶設篩選條件、看結果列表、解鎖 KOL 資料。2023 年 AI Search 上線後，用自然語言取代了部分篩選操作，但本質上還是「搜尋 → 結果列表 → 手動挑選」的流程。

40925（內部代號）要解決的問題是：**能不能讓 AI 直接幫用戶完成整個分析工作流程？**

不是搜尋完給你一個列表讓你自己看，而是：
- 多輪對話釐清需求
- 自動搜尋、篩選、比較
- 生成分析報告（PPT / PDF）
- 記住你的偏好，下次更精準

最終目標是一個坐在 KOL Radar 資料之上的**多代理對話層**，自動化分析師的工作流程。產品對外叫 KolrChat，但內部一直用 40925 這個代號——「引起大家好奇興趣」。

---

## 五個 Phase 的架構演進

從 2025 年 1 月的 RFC 到 2026 年初，40925 跑過九個版本、用過四種 LLM Supervisor 模型、整合了 MCP 協議、做出了能自動生成 PowerPoint 報告的 PPT Agent。

<div class="mermaid">
graph TD
    P1["Phase 1<br/>架構設計<br/>2025-01~03"]:::p1 --> P2["Phase 2<br/>多代理擴展<br/>2025-04~06"]:::p2
    P2 --> P3["Phase 3<br/>功能深化<br/>2025-07~09"]:::p3
    P3 --> P4["Phase 4<br/>記憶與協議遷移<br/>2025-10~12"]:::p4
    P4 --> P5["Phase 5<br/>模型現代化<br/>2026-01+"]:::p5

    P1 -.- F1["自建 Multi-Agent Core<br/>LangFuse 觀測"]:::feat
    P2 -.- F2["PPT Agent v1<br/>4 種協調模式<br/>Experiment Manager v1.0"]:::feat
    P3 -.- F3["Artifacts 持久化<br/>Deep Research Multi-hop<br/>Dynamic MCP"]:::feat
    P4 -.- F4["Mem0 記憶系統<br/>MCP SSE → Streamable HTTP<br/>GPT-4.1 → Claude Sonnet 4.5"]:::feat
    P5 -.- F5["Haiku 4.5 Supervisor<br/>個人化回應風格"]:::feat

    classDef p1 fill:#e74c3c,stroke:#c0392b,color:#fff
    classDef p2 fill:#f39c12,stroke:#e67e22,color:#fff
    classDef p3 fill:#3498db,stroke:#2980b9,color:#fff
    classDef p4 fill:#9b59b6,stroke:#8e44ad,color:#fff
    classDef p5 fill:#2ecc71,stroke:#27ae60,color:#fff
    classDef feat fill:#f5f5f5,stroke:#999
</div>

### Phase 1：架構設計（2025-01 ~ 2025-03）

先花兩個月評估框架。比較了 Dify、Oneness-AI 和自建方案，最後決定自建 Multi-Agent Core。原因有二：需要深度整合 KOL Radar 的 MCP 資料查詢，以及 Agent session 需要跟自有用戶系統打通。

同期部署 LangFuse 作為觀測平台——這個決策後來證明非常關鍵，沒有 tracing，多代理系統的 debug 幾乎不可能。

### Phase 2：多代理擴展（2025-04 ~ 2025-06）

MVP 上線後迅速擴展到四種 orchestration 模式：Swarm（群體協作）、Selector-Group（意圖路由）、Round-Robin（負載均衡）、Supervisor Model（監督者調度）。

同時上線 PPT Agent v1——第一個能產出交付物的 Agent，從自然語言查詢到 PowerPoint 輸出全流程自動化。

### Phase 3：功能深化（2025-07 ~ 2025-09）

三個重點。**Artifacts** 用 KV-Store + RDB 做持久化輸出，跨 session 保存和引用。**Deep Research Phase 2** 做了 Multi-hop 研究管線：計劃 → 搜尋 → 分析 → 總結，加上 SequentialThinking 結構化推理。**Dynamic MCP tool registration** 讓 worker agents 在運行時動態註冊工具，解決不同任務需要不同工具組合的靈活性問題。

### Phase 4：記憶與協議遷移（2025-10 ~ 2025-12）

Agent 沒有記憶是最被用戶抱怨的問題。做了兩輪 Memory POC，最終整合 Mem0 + a-mem + Autogen query layer，底層用 Elasticsearch（1,536 維 embedding）做向量搜尋、AWS Neptune 做知識圖譜。記憶以 user_id / agent_id / run_id 三個維度索引。

MCP 協議從 SSE 升級到 Streamable HTTP。Supervisor 模型從 GPT-4.1 換成 Claude Sonnet 4.5。

### Phase 5：模型現代化（2026-01+）

嘗試 Haiku 4.5 作為 Supervisor（降低幻覺，使用 todos 機制），探索 Grok。個人化回應風格按網紅類型調整。

---

## 技術決策

### 框架選型：自建 vs Dify

評估了 Dify 和 Oneness-AI（Dify 的 fork），最終選擇自建。核心考量不是「能不能用」而是「整合深度」——我們需要走 MCP 協議深度整合 KOL Radar 的資料查詢，現有框架的工具整合不夠靈活。此外，session 管理需要跟自有用戶系統打通，開源框架的 auth layer 反而成為限制。

### MCP 協議演進

從 SSE 到 Streamable HTTP 的遷移，解決了多次 Bad Request 和 Timeout 的問題。MCP 協議定義了三個核心 Protobuf 服務：
- **AgentService**：`RunStream(Task) -> stream StreamOutput`，支援 text / ChatMessage / ChatMessageList 三種輸入
- **MCPCacheService**：快取 MCP 工具呼叫的 request / response
- **ArtifactService**：管理生成物的下載 URL

### Supervisor 模型選擇

從 GPT-4.1 換到 Claude Sonnet 4.5 再試 Haiku 4.5，每次切換都帶來不同的 trade-off——推論速度 vs 品質 vs 幻覺率 vs 成本。這也是建立評估系統的核心動機：沒有系統性的比較機制，模型切換只能靠直覺。

### LangFuse 觀測

Phase 1 就部署的決策。多代理系統的每一次對話可能涉及多個 agent、多次工具呼叫、多層 context 傳遞，沒有 tracing 等於盲人開車。

---

## 評估系統

這是我認為 40925 做得最紮實的部分。

<div class="mermaid">
graph LR
    subgraph 離線評估
        EXP["Experiment Manager CLI"] --> RUN["run<br/>平行執行多組配置"]
        RUN --> JUDGE["judge<br/>批次 LLM 評分"]
        JUDGE --> RPT["report<br/>quality / latency / tool_usage"]
    end

    subgraph 線上比較
        PAIR["Pairwise Comparison"] --> TS["Thompson Sampling<br/>Dueling Bandits"]
        TS --> CI["Confidence Interval<br/>統計顯著性判定"]
    end

    subgraph 品質保證
        DE["Direct Evaluation<br/>給分"] --> CONV["分數轉換<br/>identity / linear / binary"]
    end

    RPT --> DECISION["模型選擇決策"]
    CI --> DECISION
    CONV --> DECISION

    classDef blue fill:#3498db,stroke:#2980b9,color:#fff
    classDef green fill:#2ecc71,stroke:#27ae60,color:#fff
    classDef purple fill:#9b59b6,stroke:#8e44ad,color:#fff
    class EXP,RUN,JUDGE,RPT blue
    class PAIR,TS,CI green
    class DE,CONV purple
</div>

**LLM-as-a-Judge** 用兩種模式：Direct Evaluation（給分）和 Pairwise Comparison（比較兩組輸出）。分數轉換支援 identity、linear、binary 三種方式。

**Dueling Bandits** 是更進階的模型比較機制。透過人類標記者做 pairwise comparison，用 Thompson Sampling 自動平衡 exploration 和 exploitation：

- Phase 1（Bootstrap，模型 < 5 comparisons）：優先標記數據最少的 pair
- Phase 2（Thompson Sampling）：每個模型從 Beta(wins+1, losses+1) 取樣勝率，選 combined score 最高的 pair
- 50% 隨機 swap 防位置偏差
- 統計顯著性用 confidence interval 判定：model A 的 CI 下界 > model B 的 CI 上界，判定 A 顯著優於 B

**Experiment Manager CLI** 提供三個指令：`run`（平行執行多組 agent 配置）、`judge`（批次 LLM 評分）、`report`（quality / latency / tool_usage 維度分析）。這讓每次 Supervisor 模型切換或 prompt 調整都有數據支撐，而非靠感覺。

---

## 商業化

2025 年 12 月開始密集的商業化工程：

| 項目 | 說明 |
|------|------|
| 金流 | Stripe 串接、Checkout Session API |
| 訂閱方案 | 多層級訂閱制 |
| Credit 系統 | 用量計費、推薦碼機制 |
| Admin 後台 | Next.js 15 + React 19 + shadcn/ui（palette-admin.kolr.ai） |
| 管理功能 | 用戶管理、方案管理、Credit 紀錄、Feature Card |

Admin 後台提供完整的營運視角：誰在用、用了多少 credit、哪個方案、什麼時候到期。這些資訊讓 PM 可以做數據驅動的定價調整，而非猜測。

---

## 客戶驗證

40925 的 Agent 基礎設施被用在兩個外部企業專案，這是 AI Lab 首次重要的外部企業 API 交付。

<div class="mermaid">
graph LR
    CORE["40925 Agent Core<br/>Multi-Agent + MCP + LangFuse"]:::core

    CORE --> CB["Client B<br/>工業文件解析 API"]:::cb
    CORE --> CC["Client C<br/>健康管理 LINE Bot"]:::cc

    CB --> CB_R["F1 95% / $0.03 per doc<br/>MinerU + GPT-4o VLM<br/>成功結案"]:::success
    CC --> CC_R["Acceptance 72.4/70<br/>GPT 4.1 推論<br/>通過驗收 → 不續約"]:::warn

    classDef core fill:#6366f1,color:#fff
    classDef cb fill:#10b981,color:#fff
    classDef cc fill:#f59e0b,color:#fff
    classDef success fill:#059669,color:#fff
    classDef warn fill:#ef4444,color:#fff
</div>

### Client B（製造業）：工業文件解析

把工業技術手冊（PDF）自動解析成結構化資料。Phase 1 用 MinerU 做傳統 NLP，Phase 2 引入 GPT-4o 處理複雜表格和圖片。最終 F1 > 90%，單文件成本 ~$0.03 美元。這個成本數字成為後續客戶報價的基準線。前後約 12 個月，成功結案。

### Client C（健康管理）：LINE 聊天機器人

直接用 40925 Agent 基礎設施作為底層，推論模型用 GPT 4.1。Acceptance score 72.4（門檻 70），通過驗收。但客戶最終決定不續約——不是技術問題，是客戶端的商業決策。

**教訓**：技術達標不代表商業價值被認可。「通過驗收」和「客戶願意續約」是兩件事。如果重來，我會在專案中期就開始做 business review——不只追蹤技術指標，也追蹤使用率、stakeholder 滿意度、續約意向。

兩個案子共同驗證了一件事：40925 Agent 架構在不同場景（工業文件、健康問答）是可複用的，底層的 LLM 調用、prompt 管理、評估框架可以共享。

---

## 成果與學習

**九個版本、不到一年**——從單代理 MVP 到完整 Agent 平台。

### 真實踩過的坑

| 問題 | 說明 |
|------|------|
| Report 生成時間長 | 至少等 10 分鐘，思考時間 >= 2 分鐘。Multi-hop 研究管線的代價 |
| 模型記憶遺失 | Memory 系統上線後改善，但 context window 限制仍造成資訊遺失 |
| MCP 連線穩定性 | 多次 Bad Request / Timeout，遷移到 Streamable HTTP 後穩定 |
| Token 超標 | Context Window 延伸問題、4.x MB 檔案上傳導致 OOM |
| 多語系不一致 | 用戶用中文問，Agent 有時用英文回 |
| 時間認知錯亂 | 模型訓練資料和實際時間衝突 |

### 最大的學習

Agent 系統的難度不在「讓 LLM 生成回答」，而在**工程化**——記憶持久化、工具動態註冊、多代理協調、品質評估、觀測追蹤。每一個都是獨立的工程問題，組合在一起的複雜度是**乘法不是加法**。

PM 在 ML 團隊的角色也在這個過程中被重新定義。PM 習慣用 Sprint 思考（兩週一個週期、有 AC、有上線日期），ML 工程師習慣用實驗思考（這個 config 跑完要三天，結果好才繼續，結果不好就換方向）。前者的時間是線性的，後者是條件分支的。PM 能做的最重要的事，是讓商業端理解研究需要不確定性被容忍，而不是把外部的確定性壓力直接傳到 ML 團隊身上。

AI 系統的交付日期不能從「功能設計完成」算起，要從「評估框架和部署基礎設施準備好」算起。40925 從 2024 年 12 月的 RFC 到 2025 年 4 月才有 Multi-Agent production 版本，中間 PM spec 兩個月就寫完了，但工程需要確認 AutoGen 框架穩定性、建立 LangFuse tracing、跑完 Experiment Manager v1.0、調整四種協調模式的切換邏輯。

### 技術棧總覽

| 層級 | 技術 |
|------|------|
| LLM Provider | Azure OpenAI (GPT-4.1, o3-mini)、Anthropic (Claude Sonnet 4.5, Haiku 4.5) |
| Agent Framework | 自建 Multi-Agent Core（Go + gRPC / Protobuf） |
| Observability | LangFuse |
| Memory | Mem0 + a-mem + Autogen query layer（ES + Neptune） |
| Tool Protocol | MCP (Streamable HTTP) |
| Web Search | Serper + Perplexica (LangChain + SearxNG) |
| Frontend | Shadcn UI |
| Payment | Stripe |
| Evaluation | Experiment Manager + LLM-as-a-Judge + Dueling Bandits |
| Database | PostgreSQL (Goose migration, `agentic` schema) |
| Admin | Next.js 15 + React 19 (palette-admin.kolr.ai) |

---

## 相關文章

- [Kolr Palette：獨立 AI Agent 找網紅系統的架構設計](/2026/03/10/11-kolr-palette/)
- [AI Lab 首次企業交付：從 API 到 LINE 聊天機器人](/2026/03/11/12-企業交付/)
- [AI 功能怎麼排優先級：從 P0 到 P3 的取捨邏輯](/2026/04/04/37-ai-roadmap-決策/)
- [PM 和 ML 工程師的翻譯層：我在 AI Lab 學到的溝通方式](/2026/04/05/38-pm-ml-翻譯層/)
- [一個 SaaS 產品的七年：從內部工具到全球化 AI 平台](/2026/03/07/08-saas產品七年/)
