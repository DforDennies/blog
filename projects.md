---
layout: default
title: Projects
permalink: /projects/
---

<div class="container">
  <div class="article-header">
    <h1>Projects & Spec Documents</h1>
    <p style="color:#6b6b6b;font-size:18px;margin-top:8px;">獨立開發的工具 + 真實交付的規格文件（脫敏版）</p>
  </div>

  <div class="projects-page">

  <h2 class="section-title">自建工具</h2>
  <p class="section-desc">我相信 PM 要能親手碰數據。以下是我在職期間獨立開發的工具。</p>

  <div class="project-card">
    <div class="project-badge badge-green">數據分析</div>
    <h3>Amplitude 互動式儀表板</h3>
    <p>處理 144M 事件、27,898 workspace，計算 8 維健康評分 + 全局 Benchmark。支援公司級鑽取、趨勢分析、風險監測。</p>
    <div class="project-tech">Next.js · Neon PostgreSQL · Prisma · Chart.js</div>
    <img src="{{ site.baseurl }}/assets/amplitude-dashboard.png" alt="Amplitude Dashboard 截圖" class="project-screenshot">
  </div>

  <div class="project-card">
    <div class="project-badge badge-blue">策略分析</div>
    <h3>深度研究報告：五大發現</h3>
    <p>8 章節顧問級分析 — 獲客停擺 / 年約零流失 / M3 死亡門檻 / 業務集中 / 93 家可挽回。6 個 AI Agent 平行分析產出。</p>
    <div class="project-tech">Python · 6 AI Agents · Plotly · GitHub Pages</div>
  </div>

  <div class="project-card">
    <div class="project-badge badge-orange">銷售營運</div>
    <h3>銷售報表 Pipeline</h3>
    <p>1,123 筆交易 → 9 Tab 報表 + GRR/NRR/MRR 自動追蹤。與業務手動報表 $0 差異，日/週/月報自動推送 Slack。</p>
    <div class="project-tech">Python · gspread · GitHub Actions CI</div>
  </div>

  <div class="project-card">
    <div class="project-badge badge-purple">基礎設施</div>
    <h3>Ops Dashboard</h3>
    <p>個人指揮中心 — 整合所有 Pipeline 監控、報告歸檔、系統健康狀態於一處。</p>
    <div class="project-tech">Next.js · Vercel</div>
    <img src="{{ site.baseurl }}/assets/ops-dashboard.png" alt="Ops Dashboard 截圖" class="project-screenshot">
  </div>

  <div class="project-card">
    <div class="project-badge badge-orange">流量分析</div>
    <h3>GA4 流量歸因框架</h3>
    <p>9 Key Events × 10 管道 × 5 層報告（日/週/月/季/年）+ 回溯機制。發現 Paid Social 無效，促成預算重分配。</p>
    <div class="project-tech">GA4 API · Chart.js · 健康評分模型</div>
  </div>

  <hr style="margin:48px 0;border:none;border-top:1px solid rgba(0,0,0,.08);">

  <h2 class="section-title">規格文件範例</h2>
  <p class="section-desc">PM 的核心產出之一是文件。以下是脫敏後的真實規格文件。</p>

  <div class="spec-grid">
    <a href="{{ site.baseurl }}/specs/SoW-GA貼標優化-脫敏版/" class="spec-card">
      <div class="spec-type">SoW</div>
      <h3>GA 自定義貼標優化</h3>
      <p>金融客戶 · GCP 元件規劃 · 驗收標準 · 變更管理</p>
    </a>

    <a href="{{ site.baseurl }}/specs/SoW-AI-User-Manual-脫敏版/" class="spec-card">
      <div class="spec-type">SoW</div>
      <h3>AI User Manual</h3>
      <p>製造業客戶 · AI 模型三階段交付 · WBS 時程</p>
    </a>

    <a href="{{ site.baseurl }}/specs/PRD-社群爬蟲系統-脫敏版/" class="spec-card">
      <div class="spec-type">PRD</div>
      <h3>社群爬蟲系統</h3>
      <p>Executive Summary → User Stories → Success Metrics</p>
    </a>

    <a href="{{ site.baseurl }}/specs/Tech-Spec-社群爬蟲系統-脫敏版/" class="spec-card">
      <div class="spec-type">Tech Spec</div>
      <h3>社群爬蟲系統</h3>
      <p>系統架構圖 → 資料流 → 錯誤處理 → DB Schema</p>
    </a>

    <a href="{{ site.baseurl }}/specs/Feature-Spec-全球搜尋-脫敏版/" class="spec-card">
      <div class="spec-type">Feature Spec</div>
      <h3>全球網紅搜尋</h3>
      <p>Problem Alignment → AC → Tracking 埋點設計</p>
    </a>

    <a href="{{ site.baseurl }}/specs/Data-Spec-銷售報表-脫敏版/" class="spec-card">
      <div class="spec-type">Data Spec</div>
      <h3>銷售報表規格書</h3>
      <p>17 欄定義 → 來源邏輯 → 邊界案例 → 增量更新</p>
    </a>
  </div>

  <hr style="margin:48px 0;border:none;border-top:1px solid rgba(0,0,0,.08);">

  <h2 class="section-title">技術文章</h2>
  <p class="section-desc">52 篇深度文章，涵蓋 AI 產品策略、ML 技術、數據驅動決策、平台架構。</p>
  <a href="{{ site.baseurl }}/" class="btn-articles">→ 閱讀文章</a>

  </div>
</div>
