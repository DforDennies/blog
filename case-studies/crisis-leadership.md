---
layout: page
title: "Case Study: Crisis Leadership — 57% 裁員下的產品存活戰"
permalink: /case-studies/crisis-leadership/
---

<style>
.case-study-section { margin-bottom: 2.5rem; }
.case-study-section h2 { border-bottom: 2px solid #3b82f6; padding-bottom: 0.4rem; }
.metric-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(180px, 1fr)); gap: 1rem; margin: 1.5rem 0; }
.metric-card { background: #f8fafc; border-left: 4px solid #3b82f6; padding: 1rem; border-radius: 0 8px 8px 0; }
.metric-card .number { font-size: 1.8rem; font-weight: 700; color: #1e40af; }
.metric-card .label { font-size: 0.85rem; color: #64748b; margin-top: 0.25rem; }
.timeline-badge { display: inline-block; background: #dbeafe; color: #1e40af; padding: 0.15rem 0.6rem; border-radius: 12px; font-size: 0.8rem; font-weight: 600; margin-bottom: 0.5rem; }
</style>

<span class="timeline-badge">2025 Q3 — 2026 Q1</span>

## 1. 背景

我在 Platform K（B2B SaaS 網紅行銷平台）擔任 AI/ML Product Manager，原本負責 AI Lab 的 30+ ML 模型、搜尋引擎、推薦系統、Agent 產品線。產品累計 138 個 Sprint、10,301 個 feature ticket，覆蓋八個市場。

2025 上半年，三條數據 pipeline 同時亮紅燈：

- **銷售端**：月簽約 MoM -38.5%，到期客戶續約率歸零
- **GA4 端**：月流量 MoM -36.7%，Paid Search CVR 從 2.67% 崩到 0.89%
- **Amplitude 端**：月活 MoM -37.4%，流失風險從 $344 萬膨脹到 $1,400 萬

數字在講，但組織的反應是「還好啦」。

---

## 2. 危機

<div class="mermaid">
graph TD
    A["2025 Q2<br/>業務團隊最後衝刺<br/>月簽 35 筆"] --> B["2025-07<br/>SaaS 業務團隊<br/>完整離職"]
    B --> C["2025-08<br/>公司裁員 57%<br/>PM / DS / 業務全走"]
    C --> D["真空期<br/>MRR 單月蒸發 $1,049K<br/>MoM -69%"]
    D --> E["150 家客戶<br/>無人維護"]

    style A fill:#f59e0b,color:#fff,stroke:#d97706
    style B fill:#ef4444,color:#fff,stroke:#dc2626
    style C fill:#dc2626,color:#fff,stroke:#b91c1c
    style D fill:#991b1b,color:#fff,stroke:#7f1d1d
    style E fill:#7f1d1d,color:#fff,stroke:#581c87
</div>

**2025 年 7-8 月**，公司經歷 57% 裁員。離開的不只是人頭：

- **產品 PM 離職** — 新接手的 PM 是應屆新鮮人，沒有 B2B SaaS 經驗
- **DS 團隊歸零** — 數據分析能力完全斷裂
- **SaaS 業務團隊完整離開** — Pipeline 從月均 25 筆新簽降到 6-9 筆
- **150 家付費客戶** — 沒有 CSM、沒有續約機制、沒有人跟進
- **NRR 70.9%** — 每收一塊錢，一年後只剩七毛

MRR 在 9 月單月蒸發超過一百萬。New MRR 月均從 $291K 斷崖到 $77K，降幅 73%。

這不是漸進式衰退，是引擎熄火。

---

## 3. 我的行動

裁員後我主動擴展職責，從 AI Lab PM 跨到產品、銷售、研發、內部工具四個面向。

<div class="mermaid">
graph LR
    subgraph 產品面
        P1["Dashboard 建設<br/>144M 事件 → 27K 公司監控"]
        P2["PM Mentoring<br/>帶新鮮人 PM 上手"]
        P3["產品邏輯整理<br/>功能取捨框架"]
    end
    subgraph 銷售面
        S1["市場調查<br/>競品分析 + TAM 重估"]
        S2["客製化 Demo<br/>兩天內交付數據包"]
        S3["客戶數據包<br/>單一客戶完整報告"]
    end
    subgraph 研發面
        R1["Prototype #1<br/>AI Agent 推薦引擎"]
        R2["Prototype #2<br/>品牌知識庫"]
        R3["技術夥伴整合<br/>白牌部署能力"]
    end
    subgraph 內部工具
        T1["專案管理系統<br/>50 位工程師使用"]
        T2["Slack 問題收集器<br/>業務回饋 → 結構化"]
        T3["產品事件監控<br/>每日異常偵測"]
    end

    style P1 fill:#3b82f6,color:#fff
    style P2 fill:#3b82f6,color:#fff
    style P3 fill:#3b82f6,color:#fff
    style S1 fill:#10b981,color:#fff
    style S2 fill:#10b981,color:#fff
    style S3 fill:#10b981,color:#fff
    style R1 fill:#8b5cf6,color:#fff
    style R2 fill:#8b5cf6,color:#fff
    style R3 fill:#8b5cf6,color:#fff
    style T1 fill:#f59e0b,color:#fff
    style T2 fill:#f59e0b,color:#fff
    style T3 fill:#f59e0b,color:#fff
</div>

### 產品面：填補 PM 與 DS 的雙重空缺

**Dashboard 系統建設**。我一個人用 Python + SQLite 處理了 18,724 個壓縮檔、144M 筆 Amplitude 事件、27,898 個工作區的數據，建立完整的客戶健康監控系統。從 2,940 行的單檔 HTML Dashboard 演進到 Next.js + Neon PostgreSQL + Vercel 的現代架構。

這套系統提供：
- 27,913 家公司的健康評分與風險監測
- 21 類功能的月度使用趨勢
- 雙基準線 Benchmark（全體平均 vs CRM 客戶平均）
- 自動化日/週/月報，涵蓋 21 張表、33 個指標

**PM Mentoring**。接手的新 PM 是應屆畢業生。我把過去的產品決策邏輯系統化 — 不是教他「怎麼做 PM」，是把功能取捨的三個判斷框架（客戶聲量的質 vs 量、工程可行性前置評估、策略對齊）轉化成可操作的 checklist。

**產品邏輯整理**。針對 Amplitude 數據揭露的問題 — 90% 用戶搜尋但只有 7% 收藏、1.2% 開商案、提案功能使用率 0.0% — 整理出產品應聚焦「精準搜尋」而非「探索監測」的方向建議。

### 銷售面：用數據補業務缺口

業務團隊走完後，Pipeline 幾乎清空。我做的不是取代業務，是用數據降低銷售阻力。

**市場調查與 TAM 重估**。台灣網紅行銷市場規模約 60 億台幣，但願意花錢買 SaaS 工具的市場只有零頭。我把這個分析結構化，幫助團隊聚焦真正可觸及的市場。

**客製化 Demo 與數據包**。當客戶需要看到「自己的數據」才能做決策時，我可以在兩天內從 Pipeline 產出單一客戶的完整行為報告 — 搜尋模式、功能使用深度、與同類客戶的 Benchmark 對比。這成為續約談判的關鍵武器。

**流失客戶辨識**。93 家可挽回客戶、$11.3M 潛在回收金額 — 我從數據中辨識出這批「用過產品、離開了、但沒有被挽留過」的低垂果實，提供具體的優先聯繫清單。

### 研發面：為下一階段鋪路

**兩個 Prototype**。AI Agent 推薦引擎（多代理架構 + MCP 工具整合）和品牌知識庫（LLM 驅動，855 品牌 Precision 100%）。兩個都進入銷售 pipeline，作為客製化提案的差異化賣點。

**技術夥伴整合**。白牌部署能力（Domain 綁定 + Workspace 隔離 + Theme 統一），讓企業客戶以自有品牌使用平台，擴展商業模式。

### 內部工具：為 50 位工程師建基礎設施

裁員後，原有的專案管理流程斷裂。我建了三套工具：

1. **專案管理系統** — 雙層估算框架，50 位工程師日常使用
2. **Slack 問題收集器** — 把散落在各頻道的業務回饋結構化，呈現在統一的 Web 介面
3. **產品事件每日監控** — 自動偵測異常（如 ws:46318 單日 32K 事件的自動化腳本行為），即時 Slack 通報

---

## 4. 成果

<div class="metric-grid">
  <div class="metric-card">
    <div class="number">0</div>
    <div class="label">客戶流失數（介入後）</div>
  </div>
  <div class="metric-card">
    <div class="number">+30%</div>
    <div class="label">營收回升幅度</div>
  </div>
  <div class="metric-card">
    <div class="number">2</div>
    <div class="label">客製化專案落地</div>
  </div>
  <div class="metric-card">
    <div class="number">2</div>
    <div class="label">Prototype 進入銷售 Pipeline</div>
  </div>
</div>

**客戶留存**：介入後，既有付費客戶零流失。年方案流失率維持 0% — 證明產品力本身沒有問題，問題在產品以外的客戶成功機制。

**營收回穩**：從 9 月 MRR 蒸發的谷底，靠續約機制重建和客製化專案，營收回升 +30%。

**客製化專案**：2 個客製化數據專案成功交付。雖然這類專案本質是「用工程資源換短期收入」，但在業務真空期，它們維持了現金流。

**Prototype 商業化**：AI Agent 推薦引擎和品牌知識庫兩個 Prototype 進入銷售 pipeline，成為新業務的差異化提案。

**內部效率**：3 套內部工具被 50 位工程師採用，填補了裁員後的流程斷裂。

---

## 5. 關鍵學習

<div class="mermaid">
graph TD
    L1["找到 Keyman<br/>每個環節誰在做決定？"] --> L1a["辨識留下來的關鍵人物<br/>集中資源支援他們"]
    L2["能復刻工作流程<br/>人走了，流程不能斷"] --> L2a["把隱性知識系統化<br/>Dashboard / Checklist / SOP"]
    L3["主動解決問題<br/>不等被指派"] --> L3a["危機中的職責邊界是模糊的<br/>哪裡有缺口就補哪裡"]

    style L1 fill:#1e40af,color:#fff
    style L2 fill:#7c3aed,color:#fff
    style L3 fill:#059669,color:#fff
    style L1a fill:#dbeafe,color:#1e40af
    style L2a fill:#ede9fe,color:#7c3aed
    style L3a fill:#d1fae5,color:#059669
</div>

**找到 Keyman**。裁員後最先要做的不是「補人」，是找出每個環節裡還在的關鍵人。兩位資深業務撐住了 70-80% 的營收。辨識並集中支援這些人，比平均分配資源有效得多。

**能復刻工作流程**。人可以走，但流程不能斷。DS 離職後，數據分析能力歸零 — 除非有人把它系統化。我用 Pipeline 把 27 個月的數據追蹤自動化，讓分析能力不再依賴特定個人。這個教訓也適用於 PM：把決策邏輯寫成框架，新人才能接手。

**主動解決問題**。危機中沒有「這不是我的工作」。AI Lab PM 去做銷售數據包、去帶新人 PM、去建專案管理工具 — 這些都不在原本的 job description 裡。但當組織 57% 的人不在了，職責邊界本身就不存在了。

---

## 一句話總結

> 當 57% 的同事離開，我選擇留下來，用數據填補人力缺口，用系統取代人力依賴，在六個月內把一個正在失血的產品穩住。

---

## 相關文章

- [一個 SaaS 產品的七年：從內部工具到全球化 AI 平台]({% post_url 2026-03-07-08-saas產品七年 %}) — 產品完整演進脈絡
- [MRR 單月蒸發一百萬]({% post_url 2026-04-08-s1-ep3-2025最後的榮景到斷崖 %}) — 2025 年業績斷崖的完整數據故事
- [當數據一直在講，但沒有人聽]({% post_url 2026-04-09-s1-ep4-數據沒人聽 %}) — 數據驅動決策的組織挑戰
- [AI 產品的不確定性管理]({% post_url 2026-04-02-35-不確定性管理 %}) — 62 起生產事故與品質閘門
- [從 144M 事件到互動式儀表板]({% post_url 2026-03-18-19-amplitude-dashboard %}) — Dashboard 系統的技術細節
