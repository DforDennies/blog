---
layout: page
title: "Product Requirements Document (PRD)"
permalink: /specs/PRD-社群爬蟲系統-脫敏版/
---

# Threads 學術研究爬蟲系統

---

| 文件資訊 | |
|---------|---|
| **版本** | v1.0 |
| **日期** | 2026-02-12 |
| **狀態** | Draft |
| **專案代號** | threads-academic-crawler |

---

## 1. Executive Summary

### 1.1 Problem Statement
學術研究人員需要收集 Threads 社群媒體平台上的公開貼文與互動資料，以進行社群行為分析、輿情研究、傳播學研究等學術用途。目前缺乏一個可靠、結構化且支援斷點續爬的資料收集工具。

### 1.2 Proposed Solution
建立一個基於 Python 的自動化爬蟲系統，能夠：
- 透過關鍵字搜尋發現相關議題的貼文
- 自動識別並追蹤發文用戶
- 深度爬取用戶的完整公開貼文歷史與留言互動
- 將資料結構化儲存於 SQLite 資料庫，便於後續分析

### 1.3 Key Benefits
- **自動化收集**：減少人工複製貼上的時間成本
- **結構化資料**：便於統計分析與資料視覺化
- **斷點續爬**：支援長時間大規模資料收集
- **雙軌登入**：彈性支援登入與未登入兩種模式

### 1.4 Success Metrics

| 指標 | 目標 | 衡量方式 |
|-----|------|---------|
| 資料完整性 | >95% | 成功爬取的貼文/嘗試爬取的貼文 |
| 系統穩定性 | >99% | 無崩潰運行時間/總運行時間 |
| 爬取效率 | >100 posts/hour | 每小時爬取貼文數量 |
| 資料準確性 | 100% | 欄位解析正確率 |

---

## 2. Problem Definition

### 2.1 Target Users (使用者畫像)

**主要用戶：學術研究人員**
- 傳播學、社會學、資訊科學研究者
- 需要收集社群媒體資料進行內容分析
- 具備基本 Python 操作能力
- 熟悉命令列介面

**使用情境：**
- 研究特定議題（如：選舉、社會運動）在社群媒體的傳播
- 分析特定群體的發言行為與互動模式
- 建立語料庫供自然語言處理研究使用

### 2.2 Current Pain Points (痛點分析)

| 痛點 | 嚴重度 | 說明 |
|-----|-------|-----|
| 缺乏官方 API | 高 | Threads API 功能有限，不支援搜尋 |
| 手動收集耗時 | 高 | 大規模資料收集需要數百小時人工 |
| 資料格式不一 | 中 | 手動複製的資料缺乏結構化 |
| 無法追蹤進度 | 中 | 中斷後需從頭開始 |

### 2.3 Why Now? (為何現在)
- Threads 已成為重要社群平台，月活躍用戶超過 2 億
- 學術界對 Threads 的研究需求日增
- 官方 API 限制多，無法滿足學術研究需求

---

## 3. Solution Overview

### 3.1 System Architecture (系統架構)

```
┌──────────────────────────────────────────────────────────────────────────┐
│                              CLI Interface                               │
│                              (main.py)                                   │
└───────────────────────────────────┬──────────────────────────────────────┘
                                    │
        ┌───────────────────────────┼───────────────────────────┐
        ▼                           ▼                           ▼
┌───────────────┐         ┌───────────────┐         ┌───────────────────┐
│   Scheduler   │         │  Task Queue   │         │   State Manager   │
│ (排程控制器)   │         │  (任務佇列)    │         │   (狀態管理器)     │
└───────┬───────┘         └───────┬───────┘         └─────────┬─────────┘
        │                         │                           │
        └─────────────────────────┼───────────────────────────┘
                                  │
        ┌─────────────────────────┼─────────────────────────┐
        ▼                         ▼                         ▼
┌───────────────┐         ┌───────────────┐         ┌───────────────┐
│    Search     │         │     User      │         │     Post      │
│   Crawler     │         │   Crawler     │         │   Crawler     │
└───────┬───────┘         └───────┬───────┘         └───────┬───────┘
        │                         │                         │
        └─────────────────────────┼─────────────────────────┘
                                  │
                                  ▼
                    ┌─────────────────────────┐
                    │    Browser Manager      │
                    │   (Playwright 控制器)   │
                    └───────────┬─────────────┘
                                │
                    ┌───────────┴─────────────┐
                    ▼                         ▼
            ┌───────────────┐         ┌───────────────┐
            │  Rate Limiter │         │    Parser     │
            └───────────────┘         └───────────────┘
                                              │
                                              ▼
                                ┌─────────────────────────┐
                                │      Database Layer     │
                                │   (SQLite + SQLAlchemy) │
                                └─────────────────────────┘
```

### 3.2 Core Data Flow (核心資料流)

```
Phase 1                Phase 2              Phase 3              Phase 4
┌──────────┐          ┌──────────┐        ┌──────────┐        ┌──────────┐
│ 關鍵字   │   ───▶   │ 發現     │  ───▶  │ 爬取     │  ───▶  │ 爬取     │
│ 搜尋     │          │ 用戶     │        │ 貼文     │        │ 留言     │
└──────────┘          └──────────┘        └──────────┘        └──────────┘
     │                     │                   │                   │
     ▼                     ▼                   ▼                   ▼
┌──────────┐          ┌──────────┐        ┌──────────┐        ┌──────────┐
│search_   │          │  users   │        │  posts   │        │ comments │
│logs      │          │  table   │        │  table   │        │  table   │
└──────────┘          └──────────┘        └──────────┘        └──────────┘
```

---

## 4. Functional Requirements (功能需求)

### 4.1 User Stories

#### US-001: 關鍵字搜尋
```
As a 研究人員
I want to 輸入關鍵字搜尋 Threads 上的相關貼文
So that 我可以發現研究議題相關的發文者

Acceptance Criteria:
- [ ] 支援單一或多個關鍵字搜尋
- [ ] 支援 hashtag (#標籤) 搜尋
- [ ] 自動滾動載入更多結果
- [ ] 記錄搜尋歷史與結果數量
- [ ] 自動去重發現的用戶
```

#### US-002: 用戶資料爬取
```
As a 研究人員
I want to 爬取發現用戶的個人資料
So that 我可以了解發文者的基本資訊

Acceptance Criteria:
- [ ] 取得用戶名稱、顯示名稱、簡介
- [ ] 取得粉絲數、追蹤數、貼文數
- [ ] 識別認證狀態與私人帳號
- [ ] 記錄從哪個關鍵字發現該用戶
```

#### US-003: 貼文爬取
```
As a 研究人員
I want to 爬取用戶的所有公開貼文
So that 我可以分析其發文內容與互動數據

Acceptance Criteria:
- [ ] 爬取貼文文字內容
- [ ] 記錄媒體類型與 URL（不下載）
- [ ] 取得按讚、留言、轉發數量
- [ ] 記錄發文時間
- [ ] 支援爬取全部或指定數量
```

#### US-004: 留言爬取
```
As a 研究人員
I want to 爬取貼文的留言
So that 我可以分析互動內容與對話脈絡

Acceptance Criteria:
- [ ] 每篇貼文爬取最多 100 則留言（可設定）
- [ ] 取得留言者資訊與內容
- [ ] 支援巢狀回覆結構
- [ ] 新發現的留言者自動加入用戶清單
```

#### US-005: 斷點續爬
```
As a 研究人員
I want to 中斷後能從上次進度繼續爬取
So that 我不需要每次都從頭開始

Acceptance Criteria:
- [ ] 記錄每個用戶的爬取狀態
- [ ] 記錄每篇貼文的留言爬取狀態
- [ ] 程式重啟自動從待處理任務繼續
- [ ] 提供進度查詢命令
```

#### US-006: 資料匯出
```
As a 研究人員
I want to 將爬取資料匯出為 CSV/JSON
So that 我可以使用其他工具進行分析

Acceptance Criteria:
- [ ] 支援 CSV 格式匯出
- [ ] 支援 JSON 格式匯出
- [ ] 可選擇匯出部分或全部資料
- [ ] 匯出檔案包含所有欄位
```

### 4.2 Functional Requirements Matrix

| ID | 功能需求 | 優先級 | 模組 |
|----|---------|-------|------|
| FR-001 | 關鍵字搜尋 | P0 | search.py |
| FR-002 | Hashtag 搜尋 | P1 | search.py |
| FR-003 | 用戶資料爬取 | P0 | user.py |
| FR-004 | 貼文爬取（全部） | P0 | post.py |
| FR-005 | 留言爬取（限 100 則） | P0 | comment.py |
| FR-006 | 斷點續爬 | P0 | state_manager.py |
| FR-007 | 登入模式 | P1 | auth.py |
| FR-008 | 未登入模式 | P0 | browser.py |
| FR-009 | CSV 匯出 | P0 | export.py |
| FR-010 | JSON 匯出 | P1 | export.py |
| FR-011 | 進度查詢 | P1 | main.py |
| FR-012 | 設定管理 | P2 | config.py |

### 4.3 Non-Functional Requirements

| 類別 | 需求 | 規格 |
|-----|------|-----|
| **效能** | 請求間隔 | 3-6 秒隨機延遲 |
| **效能** | 頁面載入超時 | 30 秒 |
| **穩定性** | 重試機制 | 最多 3 次，指數退避 |
| **穩定性** | 錯誤恢復 | 自動記錄失敗任務 |
| **安全性** | 反偵測 | UA 輪換、隱藏 webdriver |
| **可維護性** | 日誌記錄 | 完整操作日誌 |
| **可擴展性** | 模組化設計 | 可獨立替換各爬蟲模組 |

---

## 5. Data Model (資料模型)

### 5.1 Entity Relationship Diagram

```
┌─────────────────┐       ┌─────────────────┐       ┌─────────────────┐
│   search_logs   │       │      users      │       │      posts      │
├─────────────────┤       ├─────────────────┤       ├─────────────────┤
│ id (PK)         │       │ id (PK)         │◀──┐   │ id (PK)         │
│ keyword         │       │ username (UQ)   │   │   │ post_id (UQ)    │
│ search_type     │       │ user_id (UQ)    │   │   │ post_code (UQ)  │
│ posts_found     │       │ display_name    │   └───│ user_id (FK)    │
│ users_discovered│       │ bio             │       │ content         │
│ status          │       │ followers_count │       │ media_type      │
│ searched_at     │       │ following_count │       │ likes_count     │
└─────────────────┘       │ is_verified     │       │ replies_count   │
                          │ crawl_status    │       │ posted_at       │
                          │ discovered_from │       │ comments_crawled│
                          └─────────────────┘       └────────┬────────┘
                                   │                         │
                                   │    ┌────────────────────┘
                                   │    │
                                   ▼    ▼
                          ┌─────────────────┐
                          │    comments     │
                          ├─────────────────┤
                          │ id (PK)         │
                          │ comment_id (UQ) │
                          │ post_id (FK)    │
                          │ user_id (FK)    │
                          │ content         │
                          │ likes_count     │
                          │ parent_comment_id│
                          │ commented_at    │
                          └─────────────────┘
```

### 5.2 Table Specifications

#### users 表
| 欄位 | 類型 | 必填 | 說明 |
|-----|------|-----|------|
| id | INTEGER | Y | 主鍵，自動遞增 |
| username | TEXT | Y | 用戶名稱（唯一） |
| user_id | TEXT | N | Threads 內部 ID |
| display_name | TEXT | N | 顯示名稱 |
| bio | TEXT | N | 個人簡介 |
| profile_pic_url | TEXT | N | 頭像 URL |
| followers_count | INTEGER | N | 粉絲數 |
| following_count | INTEGER | N | 追蹤數 |
| threads_count | INTEGER | N | 貼文總數 |
| is_verified | BOOLEAN | N | 是否認證 |
| is_private | BOOLEAN | N | 是否私人帳號 |
| discovered_from_keyword | TEXT | N | 發現來源關鍵字 |
| crawl_status | TEXT | Y | pending/in_progress/completed/failed |
| last_crawled_at | DATETIME | N | 最後爬取時間 |
| created_at | DATETIME | Y | 建立時間 |
| updated_at | DATETIME | Y | 更新時間 |

#### posts 表
| 欄位 | 類型 | 必填 | 說明 |
|-----|------|-----|------|
| id | INTEGER | Y | 主鍵 |
| post_id | TEXT | Y | Threads 貼文 ID（唯一） |
| post_code | TEXT | N | URL 短碼 |
| user_id | INTEGER | Y | FK → users.id |
| content | TEXT | N | 貼文內容 |
| media_type | TEXT | N | text/image/video/carousel |
| media_count | INTEGER | N | 媒體數量 |
| media_urls | TEXT | N | JSON array |
| likes_count | INTEGER | N | 按讚數 |
| replies_count | INTEGER | N | 留言數 |
| reposts_count | INTEGER | N | 轉發數 |
| is_reply | BOOLEAN | N | 是否為回覆貼文 |
| reply_to_post_id | TEXT | N | 回覆的原貼文 ID |
| post_url | TEXT | N | 貼文 URL |
| posted_at | DATETIME | N | 發文時間 |
| scraped_at | DATETIME | Y | 爬取時間 |
| comments_crawled | BOOLEAN | Y | 留言是否已爬取 |

#### comments 表
| 欄位 | 類型 | 必填 | 說明 |
|-----|------|-----|------|
| id | INTEGER | Y | 主鍵 |
| comment_id | TEXT | Y | 留言 ID（唯一） |
| post_id | INTEGER | Y | FK → posts.id |
| user_id | INTEGER | N | FK → users.id |
| commenter_username | TEXT | N | 留言者用戶名（冗餘） |
| commenter_display_name | TEXT | N | 留言者顯示名（冗餘） |
| content | TEXT | N | 留言內容 |
| likes_count | INTEGER | N | 按讚數 |
| parent_comment_id | TEXT | N | 父留言 ID |
| reply_depth | INTEGER | N | 巢狀深度 |
| commented_at | DATETIME | N | 留言時間 |
| scraped_at | DATETIME | Y | 爬取時間 |

---

## 6. Technical Specifications (技術規格)

### 6.1 Technology Stack

| 層級 | 技術選擇 | 版本 | 選擇理由 |
|-----|---------|-----|---------|
| 語言 | Python | 3.11+ | 爬蟲生態豐富 |
| 瀏覽器自動化 | Playwright | 1.40+ | 現代、穩定、非同步支援 |
| 資料庫 | SQLite | 3.x | 輕量、無需安裝 |
| ORM | SQLAlchemy | 2.0+ | 成熟、功能完整 |
| 設定管理 | Pydantic | 2.0+ | 類型驗證、環境變數支援 |
| CLI | Typer | 0.9+ | 現代化、自動生成 help |
| 日誌 | Loguru | 0.7+ | 簡單易用 |
| 重試機制 | Tenacity | 8.2+ | 彈性重試策略 |

### 6.2 Project Structure

```
threads_crawler/
├── main.py                     # CLI 入口
├── config.py                   # 設定管理
├── requirements.txt
├── .env.example
│
├── crawler/
│   ├── __init__.py
│   ├── base.py                 # 爬蟲基礎類別
│   ├── browser.py              # Playwright 瀏覽器管理
│   ├── search.py               # 關鍵字搜尋
│   ├── user.py                 # 用戶資料爬取
│   ├── post.py                 # 貼文爬取
│   ├── comment.py              # 留言爬取
│   └── auth.py                 # 登入認證模組
│
├── database/
│   ├── __init__.py
│   ├── models.py               # SQLAlchemy ORM 模型
│   ├── db.py                   # 資料庫連線
│   └── repository.py           # 資料存取層
│
├── scheduler/
│   ├── __init__.py
│   ├── task_queue.py           # 任務佇列
│   ├── state_manager.py        # 狀態管理
│   └── scheduler.py            # 排程器
│
├── utils/
│   ├── __init__.py
│   ├── logger.py               # 日誌
│   ├── rate_limiter.py         # 頻率控制
│   ├── parser.py               # 解析工具
│   ├── anti_detect.py          # 反偵測
│   └── export.py               # 資料匯出
│
├── data/
│   ├── threads.db              # 資料庫
│   ├── sessions/               # Session 儲存
│   └── exports/                # 匯出檔案
│
└── tests/
    ├── test_parser.py
    └── test_database.py
```

### 6.3 Configuration Schema

```python
class CrawlerSettings:
    # 基本設定
    PROJECT_NAME: str = "threads_crawler"
    DEBUG: bool = False
    DATABASE_PATH: Path = "data/threads.db"

    # 瀏覽器設定
    BROWSER_HEADLESS: bool = True
    BROWSER_TIMEOUT: int = 30000

    # 登入設定
    AUTH_MODE: str = "anonymous"  # anonymous / session / credentials
    SESSION_FILE: Path = "data/sessions/session.json"

    # 頻率控制
    REQUEST_DELAY_MIN: float = 3.0
    REQUEST_DELAY_MAX: float = 6.0

    # 爬取限制
    MAX_POSTS_PER_USER: int = None  # None = 全部
    MAX_COMMENTS_PER_POST: int = 100
    MAX_SEARCH_RESULTS: int = 500

    # 重試設定
    MAX_RETRIES: int = 3
    RETRY_DELAY: float = 10.0
```

---

## 7. CLI Interface (命令列介面)

### 7.1 Command Reference

```bash
# 搜尋相關
python main.py search "關鍵字1" "關鍵字2"     # 關鍵字搜尋
python main.py search-hashtag "標籤1"         # 標籤搜尋

# 爬取相關
python main.py crawl users                     # 爬取待處理用戶
python main.py crawl posts --all               # 爬取所有用戶貼文
python main.py crawl posts --user @username    # 爬取指定用戶
python main.py crawl comments --all            # 爬取所有貼文留言

# 一鍵執行
python main.py run --keywords "關鍵字" --full  # 完整流程

# 狀態管理
python main.py status                          # 查看進度

# 資料匯出
python main.py export --format csv             # CSV 匯出
python main.py export --format json            # JSON 匯出

# 認證管理
python main.py auth login                      # 登入
python main.py auth status                     # 檢查狀態
python main.py auth logout                     # 登出
```

---

## 8. Error Handling (錯誤處理)

### 8.1 Error Types

| 錯誤類型 | 觸發條件 | 處理策略 |
|---------|---------|---------|
| NetworkError | 網路連線失敗 | 重試 3 次，指數退避 |
| RateLimitError | 被頻率限制 | 暫停 5-10 分鐘 |
| AuthenticationError | Session 過期 | 提示重新登入 |
| PageNotFoundError | 頁面不存在 | 標記跳過，記錄日誌 |
| BlockedError | IP 被封鎖 | 暫停爬取，發送通知 |
| ParseError | 解析失敗 | 記錄原始資料，跳過 |

### 8.2 Retry Strategy

```python
@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=10, max=60),
    retry=retry_if_exception_type((NetworkError, TimeoutError))
)
```

---

## 9. Anti-Detection Strategy (反偵測策略)

### 9.1 Measures

| 策略 | 實作方式 |
|-----|---------|
| User-Agent 輪換 | 隨機選擇常見瀏覽器 UA |
| 隱藏 webdriver | 覆蓋 navigator.webdriver |
| 隨機延遲 | 3-6 秒 + 抖動 |
| 模擬人類行為 | 隨機滑鼠移動、打字速度 |
| Session 管理 | 支援持久化登入狀態 |

---

## 10. Risks & Mitigations (風險與緩解)

| 風險 | 機率 | 影響 | 緩解策略 |
|-----|-----|-----|---------|
| Threads 網站結構變更 | 高 | 高 | 模組化設計，快速更新選擇器 |
| IP 被封鎖 | 中 | 高 | 控制請求頻率，支援 Proxy |
| 登入驗證碼 | 中 | 中 | 支援手動處理，Session 持久化 |
| 資料解析失敗 | 低 | 中 | 錯誤處理，記錄原始資料 |
| 法律合規風險 | 低 | 高 | 僅爬取公開資料，學術用途聲明 |

---

## 11. Development Phases (開發階段)

### Phase 1: 專案初始化 (Foundation)
**交付物：**
- [ ] 專案資料夾結構
- [ ] requirements.txt
- [ ] config.py 設定檔
- [ ] .env.example
- [ ] 資料庫 schema (models.py)

### Phase 2: 核心基礎設施 (Infrastructure)
**交付物：**
- [ ] database/db.py - 資料庫連線
- [ ] database/models.py - ORM 模型
- [ ] database/repository.py - CRUD 操作
- [ ] utils/logger.py - 日誌模組
- [ ] utils/rate_limiter.py - 頻率控制

### Phase 3: 瀏覽器與認證 (Browser & Auth)
**交付物：**
- [ ] crawler/browser.py - Playwright 管理器
- [ ] crawler/auth.py - 登入認證
- [ ] utils/anti_detect.py - 反偵測工具

### Phase 4: 爬蟲模組 (Crawlers)
**交付物：**
- [ ] crawler/search.py - 搜尋功能
- [ ] crawler/user.py - 用戶爬取
- [ ] crawler/post.py - 貼文爬取
- [ ] crawler/comment.py - 留言爬取

### Phase 5: 排程與狀態管理 (Scheduler)
**交付物：**
- [ ] scheduler/state_manager.py
- [ ] scheduler/task_queue.py
- [ ] scheduler/scheduler.py

### Phase 6: CLI 與匯出 (CLI & Export)
**交付物：**
- [ ] main.py - CLI 介面
- [ ] utils/export.py - 資料匯出

### Phase 7: 測試與優化 (Testing)
**交付物：**
- [ ] 單元測試
- [ ] 整合測試
- [ ] 效能優化

---

## 12. Dependencies (依賴套件)

```
# requirements.txt

# 核心
playwright>=1.40.0
sqlalchemy>=2.0.0
pydantic>=2.0.0
pydantic-settings>=2.0.0

# CLI
typer>=0.9.0
rich>=13.0.0

# 工具
loguru>=0.7.0
python-dotenv>=1.0.0
tenacity>=8.2.0

# 資料處理
pandas>=2.0.0
orjson>=3.9.0

# 開發
pytest>=7.4.0
pytest-asyncio>=0.21.0
```

---

## 13. Acceptance Criteria (驗收標準)

### 13.1 功能驗收

| 功能 | 驗收條件 |
|-----|---------|
| 關鍵字搜尋 | 能搜尋並返回 ≥50 則結果 |
| 用戶爬取 | 正確取得所有必要欄位 |
| 貼文爬取 | 能爬取用戶全部公開貼文 |
| 留言爬取 | 正確爬取 100 則留言 |
| 斷點續爬 | 重啟後能從中斷處繼續 |
| 資料匯出 | CSV/JSON 格式正確可用 |

### 13.2 非功能驗收

| 項目 | 驗收條件 |
|-----|---------|
| 穩定性 | 連續運行 4 小時無崩潰 |
| 錯誤處理 | 所有錯誤有日誌記錄 |
| 資料完整性 | 無資料遺漏或損壞 |

---

## 14. Appendix (附錄)

### A. Threads URL Structure

| 頁面 | URL 格式 |
|-----|---------|
| 搜尋 | `https://www.threads.net/search?q={keyword}` |
| 用戶頁 | `https://www.threads.net/@{username}` |
| 貼文頁 | `https://www.threads.net/@{username}/post/{code}` |
| 標籤頁 | `https://www.threads.net/tag/{hashtag}` |

### B. Glossary (術語表)

| 術語 | 說明 |
|-----|------|
| 爬蟲 (Crawler) | 自動化網頁資料收集程式 |
| 斷點續爬 | 中斷後能從上次進度繼續 |
| Rate Limiting | 請求頻率限制 |
| Session | 瀏覽器登入狀態 |
| ORM | Object-Relational Mapping |
