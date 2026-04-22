---
layout: page
title: "Threads 學術爬蟲系統設計（完整版）"
permalink: /specs/Tech-Spec-社群爬蟲系統-脫敏版/
---

# Threads 學術爬蟲系統設計（完整版）

## 專案目標
為學術研究目的，爬取 Threads 公開帳號的貼文和留言資料。

---

## 1. 整體流程

```
┌─────────────────────────────────────────────────────────────────┐
│                         Phase 1: 搜尋                           │
│  輸入關鍵字 → 搜尋 Threads → 取得相關貼文列表                      │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                      Phase 2: 用戶識別                          │
│  解析貼文 → 提取發文者資訊 → 建立用戶清單（去重）                   │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                      Phase 3: 深度爬取                          │
│  遍歷用戶清單 → 爬取每位用戶的：                                  │
│    - 個人資料（bio、followers、following 等）                    │
│    - 所有公開貼文                                               │
│    - 貼文的留言（每篇最多 100 則）                               │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                      Phase 4: 資料儲存                          │
│  結構化資料 → 存入 SQLite 資料庫                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. 系統架構圖

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
│  (搜尋爬蟲)    │         │  (用戶爬蟲)    │         │  (貼文爬蟲)    │
└───────┬───────┘         └───────┬───────┘         └───────┬───────┘
        │                         │                         │
        └─────────────────────────┼─────────────────────────┘
                                  │
                                  ▼
                    ┌─────────────────────────┐
                    │    Browser Manager      │
                    │   (Playwright 控制器)   │
                    │  - 登入/未登入模式切換    │
                    │  - Session 管理         │
                    │  - 反偵測機制           │
                    └───────────┬─────────────┘
                                │
                    ┌───────────┴─────────────┐
                    ▼                         ▼
            ┌───────────────┐         ┌───────────────┐
            │  Rate Limiter │         │    Parser     │
            │  (頻率控制器)  │         │   (解析器)    │
            └───────────────┘         └───────────────┘
                                              │
                                              ▼
                                ┌─────────────────────────┐
                                │      Database Layer     │
                                │   (SQLite + SQLAlchemy) │
                                └─────────────────────────┘
```

---

## 3. 資料結構設計

### 3.1 用戶表 (users)
```sql
CREATE TABLE users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    username TEXT UNIQUE NOT NULL,           -- @username
    user_id TEXT UNIQUE,                     -- Threads 內部 ID
    display_name TEXT,                       -- 顯示名稱
    bio TEXT,                                -- 個人簡介
    profile_pic_url TEXT,                    -- 頭像 URL
    followers_count INTEGER DEFAULT 0,
    following_count INTEGER DEFAULT 0,
    threads_count INTEGER DEFAULT 0,         -- 貼文總數
    is_verified BOOLEAN DEFAULT FALSE,
    is_private BOOLEAN DEFAULT FALSE,        -- 是否私人帳號
    profile_url TEXT,

    -- 爬取相關
    discovered_from_keyword TEXT,            -- 從哪個關鍵字發現
    crawl_status TEXT DEFAULT 'pending',     -- pending/in_progress/completed/failed
    last_crawled_at DATETIME,

    -- 元資料
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_crawl_status ON users(crawl_status);
```

### 3.2 貼文表 (posts)
```sql
CREATE TABLE posts (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    post_id TEXT UNIQUE NOT NULL,            -- Threads 貼文 ID
    post_code TEXT UNIQUE,                   -- URL 中的短碼 (如 CxYz123)
    user_id INTEGER NOT NULL,                -- FK → users.id

    -- 內容
    content TEXT,                            -- 貼文文字
    media_type TEXT,                         -- text/image/video/carousel
    media_count INTEGER DEFAULT 0,           -- 媒體數量
    media_urls TEXT,                         -- JSON array of URLs

    -- 互動數據
    likes_count INTEGER DEFAULT 0,
    replies_count INTEGER DEFAULT 0,
    reposts_count INTEGER DEFAULT 0,
    quotes_count INTEGER DEFAULT 0,

    -- 回覆關係
    is_reply BOOLEAN DEFAULT FALSE,
    reply_to_post_id TEXT,                   -- 如果是回覆，原貼文 ID

    -- URL
    post_url TEXT,

    -- 時間
    posted_at DATETIME,                      -- 發文時間
    scraped_at DATETIME DEFAULT CURRENT_TIMESTAMP,

    -- 爬取狀態
    comments_crawled BOOLEAN DEFAULT FALSE,  -- 留言是否已爬取

    FOREIGN KEY (user_id) REFERENCES users(id)
);

CREATE INDEX idx_posts_user_id ON posts(user_id);
CREATE INDEX idx_posts_post_id ON posts(post_id);
CREATE INDEX idx_posts_posted_at ON posts(posted_at);
CREATE INDEX idx_posts_comments_crawled ON posts(comments_crawled);
```

### 3.3 留言表 (comments)
```sql
CREATE TABLE comments (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    comment_id TEXT UNIQUE NOT NULL,         -- 留言 ID
    post_id INTEGER NOT NULL,                -- FK → posts.id
    user_id INTEGER,                         -- FK → users.id（留言者，可能為新用戶）

    -- 留言者資訊（冗餘儲存，避免額外查詢）
    commenter_username TEXT,
    commenter_display_name TEXT,
    commenter_verified BOOLEAN DEFAULT FALSE,

    -- 內容
    content TEXT,
    likes_count INTEGER DEFAULT 0,

    -- 巢狀回覆
    parent_comment_id TEXT,                  -- 如果是回覆留言
    reply_depth INTEGER DEFAULT 0,           -- 巢狀深度

    -- 時間
    commented_at DATETIME,
    scraped_at DATETIME DEFAULT CURRENT_TIMESTAMP,

    FOREIGN KEY (post_id) REFERENCES posts(id),
    FOREIGN KEY (user_id) REFERENCES users(id)
);

CREATE INDEX idx_comments_post_id ON comments(post_id);
CREATE INDEX idx_comments_user_id ON comments(user_id);
```

### 3.4 搜尋記錄表 (search_logs)
```sql
CREATE TABLE search_logs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    keyword TEXT NOT NULL,
    search_type TEXT DEFAULT 'keyword',      -- keyword/hashtag/user
    posts_found INTEGER DEFAULT 0,
    users_discovered INTEGER DEFAULT 0,
    status TEXT DEFAULT 'completed',         -- completed/partial/failed
    error_message TEXT,
    searched_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

### 3.5 爬取任務表 (crawl_tasks)
```sql
CREATE TABLE crawl_tasks (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    task_type TEXT NOT NULL,                 -- search/user/posts/comments
    target_id TEXT,                          -- username 或 post_id
    status TEXT DEFAULT 'pending',           -- pending/in_progress/completed/failed
    priority INTEGER DEFAULT 0,              -- 優先級
    retry_count INTEGER DEFAULT 0,
    max_retries INTEGER DEFAULT 3,
    error_message TEXT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    started_at DATETIME,
    completed_at DATETIME
);

CREATE INDEX idx_crawl_tasks_status ON crawl_tasks(status);
CREATE INDEX idx_crawl_tasks_priority ON crawl_tasks(priority DESC);
```

---

## 4. 模組架構

```
threads_crawler/
├── main.py                     # CLI 入口
├── config.py                   # 設定管理
├── requirements.txt
├── .env.example                # 環境變數範本
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
│   ├── repository.py           # 資料存取層
│   └── migrations/             # 資料庫遷移（可選）
│
├── scheduler/
│   ├── __init__.py
│   ├── task_queue.py           # 任務佇列
│   ├── state_manager.py        # 狀態管理（斷點續爬）
│   └── scheduler.py            # 排程器
│
├── utils/
│   ├── __init__.py
│   ├── logger.py               # 日誌（loguru）
│   ├── rate_limiter.py         # 頻率控制
│   ├── parser.py               # HTML/JSON 解析
│   ├── anti_detect.py          # 反偵測工具
│   └── export.py               # 資料匯出
│
├── data/
│   ├── threads.db              # SQLite 資料庫
│   ├── sessions/               # 登入 session 儲存
│   └── exports/                # 匯出檔案
│
└── tests/
    ├── __init__.py
    ├── test_parser.py
    ├── test_database.py
    └── fixtures/               # 測試資料
```

---

## 5. 設定檔設計 (config.py)

```python
from pydantic_settings import BaseSettings
from typing import Optional, List
from pathlib import Path

class CrawlerSettings(BaseSettings):
    """爬蟲設定"""

    # === 基本設定 ===
    PROJECT_NAME: str = "threads_crawler"
    DEBUG: bool = False

    # === 資料庫 ===
    DATABASE_PATH: Path = Path("data/threads.db")

    # === 瀏覽器設定 ===
    BROWSER_HEADLESS: bool = True           # 無頭模式
    BROWSER_SLOW_MO: int = 100              # 操作延遲（毫秒）
    BROWSER_TIMEOUT: int = 30000            # 頁面載入超時

    # === 登入設定 ===
    AUTH_MODE: str = "anonymous"            # anonymous / session / credentials
    SESSION_FILE: Optional[Path] = Path("data/sessions/session.json")
    THREADS_USERNAME: Optional[str] = None
    THREADS_PASSWORD: Optional[str] = None

    # === 頻率控制 ===
    REQUEST_DELAY_MIN: float = 3.0          # 最小請求間隔（秒）
    REQUEST_DELAY_MAX: float = 6.0          # 最大請求間隔（秒）
    PAGE_LOAD_DELAY: float = 2.0            # 頁面載入後等待
    SCROLL_DELAY: float = 1.5               # 滾動間隔

    # === 爬取限制 ===
    MAX_POSTS_PER_USER: Optional[int] = None  # None = 全部
    MAX_COMMENTS_PER_POST: int = 100
    MAX_SEARCH_RESULTS: int = 500           # 每個關鍵字最多結果
    MAX_SCROLL_ATTEMPTS: int = 50           # 最大滾動次數

    # === 重試設定 ===
    MAX_RETRIES: int = 3
    RETRY_DELAY: float = 10.0

    # === 反偵測 ===
    RANDOMIZE_USER_AGENT: bool = True
    USER_AGENTS: List[str] = [
        "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) ...",
        "Mozilla/5.0 (Windows NT 10.0; Win64; x64) ...",
        # ... 更多 UA
    ]

    # === 日誌 ===
    LOG_LEVEL: str = "INFO"
    LOG_FILE: Path = Path("data/logs/crawler.log")

    class Config:
        env_file = ".env"
        env_prefix = "CRAWLER_"
```

---

## 6. 核心模組 API 設計

### 6.1 Browser Manager (browser.py)

```python
class BrowserManager:
    """Playwright 瀏覽器管理器"""

    async def __aenter__(self) -> "BrowserManager":
        """啟動瀏覽器"""

    async def __aexit__(self, *args) -> None:
        """關閉瀏覽器"""

    async def new_page(self) -> Page:
        """建立新分頁（含反偵測設定）"""

    async def goto(self, url: str, wait_for: str = "networkidle") -> None:
        """導航至頁面"""

    async def scroll_to_bottom(self, max_scrolls: int = 50) -> int:
        """滾動載入更多內容，回傳滾動次數"""

    async def wait_for_selector(self, selector: str, timeout: int = 30000) -> Element:
        """等待元素出現"""

    async def save_session(self, path: Path) -> None:
        """儲存登入 session"""

    async def load_session(self, path: Path) -> bool:
        """載入已儲存的 session"""
```

### 6.2 Auth Module (auth.py)

```python
class AuthManager:
    """認證管理器"""

    def __init__(self, browser: BrowserManager, config: CrawlerSettings):
        pass

    async def login(self, username: str, password: str) -> bool:
        """使用帳密登入"""

    async def login_with_session(self, session_file: Path) -> bool:
        """使用已儲存的 session 登入"""

    async def check_login_status(self) -> bool:
        """檢查是否已登入"""

    async def handle_checkpoint(self) -> bool:
        """處理安全驗證（需要人工介入）"""
```

### 6.3 Search Crawler (search.py)

```python
@dataclass
class SearchResult:
    post_id: str
    post_code: str
    content: str
    username: str
    posted_at: datetime
    likes_count: int
    replies_count: int

class SearchCrawler:
    """關鍵字搜尋爬蟲"""

    def __init__(self, browser: BrowserManager, db: Repository):
        pass

    async def search(self, keyword: str, max_results: int = 500) -> List[SearchResult]:
        """
        搜尋關鍵字，回傳貼文列表

        流程：
        1. 導航至 https://www.threads.net/search?q={keyword}
        2. 等待搜尋結果載入
        3. 滾動載入更多結果
        4. 解析每則貼文
        5. 提取用戶並存入資料庫
        """

    async def search_hashtag(self, hashtag: str) -> List[SearchResult]:
        """搜尋標籤"""

    def _parse_search_results(self, html: str) -> List[SearchResult]:
        """解析搜尋結果頁面"""
```

### 6.4 User Crawler (user.py)

```python
@dataclass
class UserProfile:
    username: str
    user_id: str
    display_name: str
    bio: str
    followers_count: int
    following_count: int
    threads_count: int
    is_verified: bool
    is_private: bool
    profile_pic_url: str

class UserCrawler:
    """用戶資料爬蟲"""

    def __init__(self, browser: BrowserManager, db: Repository):
        pass

    async def get_profile(self, username: str) -> Optional[UserProfile]:
        """
        取得用戶個人資料

        URL: https://www.threads.net/@{username}
        """

    async def crawl_pending_users(self) -> int:
        """爬取所有待處理的用戶，回傳處理數量"""

    def _parse_profile_page(self, html: str) -> UserProfile:
        """解析個人頁面"""
```

### 6.5 Post Crawler (post.py)

```python
@dataclass
class Post:
    post_id: str
    post_code: str
    content: str
    media_type: str
    media_urls: List[str]
    likes_count: int
    replies_count: int
    reposts_count: int
    posted_at: datetime
    is_reply: bool
    reply_to_post_id: Optional[str]

class PostCrawler:
    """貼文爬蟲"""

    def __init__(self, browser: BrowserManager, db: Repository):
        pass

    async def get_user_posts(
        self,
        username: str,
        max_posts: Optional[int] = None
    ) -> List[Post]:
        """
        取得用戶的所有貼文

        流程：
        1. 導航至用戶頁面
        2. 滾動載入所有貼文
        3. 解析每則貼文
        """

    async def get_post_detail(self, post_code: str) -> Optional[Post]:
        """
        取得單則貼文詳情

        URL: https://www.threads.net/@{username}/post/{post_code}
        """

    async def crawl_user_posts(self, user_id: int) -> int:
        """爬取指定用戶的所有貼文"""

    def _parse_post_element(self, element: Element) -> Post:
        """解析單則貼文元素"""
```

### 6.6 Comment Crawler (comment.py)

```python
@dataclass
class Comment:
    comment_id: str
    content: str
    username: str
    display_name: str
    is_verified: bool
    likes_count: int
    commented_at: datetime
    parent_comment_id: Optional[str]
    reply_depth: int

class CommentCrawler:
    """留言爬蟲"""

    def __init__(self, browser: BrowserManager, db: Repository):
        pass

    async def get_post_comments(
        self,
        post_code: str,
        max_comments: int = 100
    ) -> List[Comment]:
        """
        取得貼文的留言

        流程：
        1. 導航至貼文頁面
        2. 展開所有留言
        3. 解析留言（含巢狀回覆）
        4. 儲存新發現的用戶
        """

    async def crawl_uncrawled_posts(self) -> int:
        """爬取所有未爬取留言的貼文"""

    def _parse_comment_element(self, element: Element) -> Comment:
        """解析單則留言"""
```

---

## 7. 狀態管理與斷點續爬

### 7.1 爬取狀態流程

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           用戶爬取狀態                                  │
│                                                                         │
│   pending ──→ in_progress ──→ completed                                │
│      │             │              │                                     │
│      │             ▼              │                                     │
│      │          failed ←─────────┘                                     │
│      │             │                                                    │
│      └─────────────┘ (重試)                                            │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                           貼文留言爬取狀態                              │
│                                                                         │
│   posts.comments_crawled: FALSE ──→ TRUE                               │
│                                                                         │
│   每次重啟時，查詢 comments_crawled = FALSE 的貼文繼續爬取              │
└─────────────────────────────────────────────────────────────────────────┘
```

### 7.2 State Manager

```python
class StateManager:
    """狀態管理器 - 支援斷點續爬"""

    def __init__(self, db: Repository):
        pass

    def get_pending_users(self) -> List[User]:
        """取得待爬取的用戶列表"""

    def get_users_without_posts(self) -> List[User]:
        """取得尚未爬取貼文的用戶"""

    def get_posts_without_comments(self) -> List[Post]:
        """取得尚未爬取留言的貼文"""

    def mark_user_in_progress(self, user_id: int) -> None:
        """標記用戶爬取中"""

    def mark_user_completed(self, user_id: int) -> None:
        """標記用戶爬取完成"""

    def mark_user_failed(self, user_id: int, error: str) -> None:
        """標記用戶爬取失敗"""

    def get_crawl_progress(self) -> dict:
        """取得整體爬取進度統計"""
```

---

## 8. CLI 介面設計

```bash
# 顯示說明
python main.py --help

# 搜尋關鍵字
python main.py search "關鍵字1" "關鍵字2" --max-results 500

# 搜尋標籤
python main.py search-hashtag "科技" "AI"

# 爬取用戶資料（從搜尋結果發現的用戶）
python main.py crawl users --limit 100

# 爬取貼文
python main.py crawl posts --user "@username"        # 指定用戶
python main.py crawl posts --all                      # 所有待爬取用戶

# 爬取留言
python main.py crawl comments --post "CxYz123"       # 指定貼文
python main.py crawl comments --all                   # 所有待爬取貼文

# 一鍵完整爬取（搜尋 → 用戶 → 貼文 → 留言）
python main.py run --keywords "關鍵字" --full

# 查看進度
python main.py status

# 匯出資料
python main.py export --format csv --output ./exports
python main.py export --format json --output ./exports

# 登入管理
python main.py auth login                             # 互動式登入
python main.py auth status                            # 檢查登入狀態
python main.py auth logout                            # 清除 session

# 設定
python main.py config show                            # 顯示目前設定
python main.py config set REQUEST_DELAY_MIN 5        # 修改設定
```

---

## 9. Threads 網站結構分析

### 9.1 URL 結構

| 頁面 | URL 格式 |
|------|----------|
| 首頁 | `https://www.threads.net/` |
| 搜尋 | `https://www.threads.net/search?q={keyword}&serp_type=default` |
| 用戶頁 | `https://www.threads.net/@{username}` |
| 單則貼文 | `https://www.threads.net/@{username}/post/{post_code}` |
| 標籤頁 | `https://www.threads.net/tag/{hashtag}` |

### 9.2 資料取得方式

Threads 是 React SPA，資料透過 GraphQL API 載入：

```
API Endpoint: https://www.threads.net/api/graphql

Headers:
- X-FB-LSD: {token}
- X-IG-App-ID: {app_id}
- Content-Type: application/x-www-form-urlencoded
```

**爬取策略**：
1. **方案 A（推薦）**：使用 Playwright 模擬瀏覽器，等待 JavaScript 渲染完成後解析 DOM
2. **方案 B**：攔截 GraphQL 請求，直接解析 JSON 回應（更高效但更容易被封鎖）

### 9.3 DOM 選擇器（參考，可能會變動）

```python
SELECTORS = {
    # 搜尋結果
    "search_post": "[data-pressable-container='true']",
    "search_content": "[data-text-content]",

    # 用戶頁面
    "profile_name": "h1",
    "profile_bio": "[data-biography]",
    "followers_count": "[title*='followers']",

    # 貼文
    "post_content": "[data-text-content]",
    "post_time": "time",
    "post_likes": "[aria-label*='like']",
    "post_media": "img[src*='cdninstagram'], video",

    # 留言
    "comment_container": "[data-comment]",
    "comment_content": "[data-text-content]",
    "load_more_comments": "button:has-text('View more replies')",
}
```

---

## 10. 反偵測策略

### 10.1 瀏覽器指紋

```python
async def setup_anti_detect(page: Page) -> None:
    """設定反偵測"""

    # 1. 覆蓋 navigator.webdriver
    await page.add_init_script("""
        Object.defineProperty(navigator, 'webdriver', {
            get: () => undefined
        });
    """)

    # 2. 模擬真實螢幕尺寸
    await page.set_viewport_size({"width": 1920, "height": 1080})

    # 3. 設定語言和時區
    await page.context.add_cookies([...])

    # 4. 隨機化滑鼠移動
    # 5. 模擬人類打字速度
```

### 10.2 請求模式

```python
class RateLimiter:
    """頻率控制器"""

    def __init__(self, min_delay: float, max_delay: float):
        self.min_delay = min_delay
        self.max_delay = max_delay
        self.last_request = 0

    async def wait(self) -> None:
        """等待隨機時間"""
        elapsed = time.time() - self.last_request
        delay = random.uniform(self.min_delay, self.max_delay)
        if elapsed < delay:
            await asyncio.sleep(delay - elapsed)
        self.last_request = time.time()

    async def wait_with_jitter(self, base_delay: float) -> None:
        """加入抖動的等待"""
        jitter = random.uniform(-0.5, 0.5) * base_delay
        await asyncio.sleep(base_delay + jitter)
```

---

## 11. 錯誤處理

### 11.1 錯誤類型

```python
class CrawlerError(Exception):
    """爬蟲基礎錯誤"""
    pass

class RateLimitError(CrawlerError):
    """被頻率限制"""
    pass

class AuthenticationError(CrawlerError):
    """登入失敗或 session 過期"""
    pass

class PageNotFoundError(CrawlerError):
    """頁面不存在（用戶刪除、私人帳號等）"""
    pass

class NetworkError(CrawlerError):
    """網路錯誤"""
    pass

class BlockedError(CrawlerError):
    """被封鎖"""
    pass
```

### 11.2 重試策略

```python
@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=10, max=60),
    retry=retry_if_exception_type((NetworkError, TimeoutError))
)
async def fetch_with_retry(url: str) -> Response:
    """帶重試的請求"""
    pass
```

### 11.3 錯誤恢復

| 錯誤類型 | 處理方式 |
|---------|---------|
| NetworkError | 重試 3 次，指數退避 |
| RateLimitError | 暫停 5-10 分鐘後繼續 |
| AuthenticationError | 提示重新登入 |
| PageNotFoundError | 標記用戶為不可用，跳過 |
| BlockedError | 暫停爬取，發送通知 |

---

## 12. 需求確認

| 項目 | 決定 |
|------|------|
| 爬取深度 | 全部公開貼文 |
| 留言爬取 | 每篇貼文最多 100 則（可設定） |
| 媒體處理 | 只記錄 URL，不下載圖片/影片 |
| 登入模式 | **雙軌制**：支援登入與未登入兩種模式 |

---

## 13. 實作計畫

### Phase 1: 專案初始化
- [ ] 建立專案資料夾結構
- [ ] 建立 `requirements.txt`
- [ ] 建立 `config.py` 設定檔
- [ ] 建立 `.env.example`
- [ ] 建立資料庫 schema 和 ORM 模型

### Phase 2: 核心基礎設施
- [ ] `database/db.py` - 資料庫連線
- [ ] `database/models.py` - ORM 模型
- [ ] `database/repository.py` - 資料存取層
- [ ] `utils/logger.py` - 日誌模組
- [ ] `utils/rate_limiter.py` - 頻率控制

### Phase 3: 瀏覽器與認證
- [ ] `crawler/browser.py` - Playwright 管理器
- [ ] `crawler/auth.py` - 登入認證模組
- [ ] `utils/anti_detect.py` - 反偵測工具

### Phase 4: 爬蟲模組
- [ ] `crawler/search.py` - 搜尋功能
- [ ] `crawler/user.py` - 用戶爬取
- [ ] `crawler/post.py` - 貼文爬取
- [ ] `crawler/comment.py` - 留言爬取

### Phase 5: 排程與狀態管理
- [ ] `scheduler/state_manager.py` - 狀態管理
- [ ] `scheduler/task_queue.py` - 任務佇列
- [ ] `scheduler/scheduler.py` - 排程器

### Phase 6: CLI 與匯出
- [ ] `main.py` - CLI 介面
- [ ] `utils/export.py` - 資料匯出

### Phase 7: 測試與優化
- [ ] 單元測試
- [ ] 整合測試
- [ ] 效能優化

---

## 14. 依賴套件 (requirements.txt)

```
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
tenacity>=8.2.0           # 重試機制

# 資料處理
pandas>=2.0.0             # 資料匯出
orjson>=3.9.0             # 快速 JSON

# 開發
pytest>=7.4.0
pytest-asyncio>=0.21.0
```

---

## 15. 驗證方式

1. **單元測試**
   - 測試 Parser 的解析邏輯（使用 fixtures）
   - 測試 Repository 的 CRUD 操作
   - 測試 RateLimiter 的延遲機制

2. **整合測試**
   - 用測試關鍵字執行搜尋（如 "test"）
   - 爬取 1-2 個公開測試帳號
   - 驗證資料正確寫入資料庫

3. **手動驗證**
   - 檢查日誌輸出
   - 匯出 CSV 檢查資料完整性
   - 使用 `python main.py status` 查看進度
