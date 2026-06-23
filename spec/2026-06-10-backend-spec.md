# news-listen バックエンド実装仕様書 — Task 4〜13

> **対象スコープ**: `backend/` ディレクトリ内 Task 4〜13（Task 1〜3 完了済み）
> **生成日**: 2026-06-10

---

## 1. システム全体概要

| 項目 | 内容 |
|------|------|
| 実行環境 | Python 3.12 / FastAPI / Cloud Run (GCP) |
| データストア | Cloud Firestore (Native モード, asia-northeast1) |
| ファイルストレージ | Cloud Storage (`news-listen-20260610-podcasts`) |
| AI | Gemini 2.5 Flash (テキスト生成 + TTS) |
| 認証（基盤） | X-API-Key ヘッダー固定キー (Secret Manager 管理) |
| 認証（利用者） | セッションベース（bcrypt + 不透明トークン + Firestore `sessions`、admin/user ロール）。Web=httpOnly Cookie `nl_session` / iOS=`Authorization: Bearer`。設計は [ADR-013](../adr/013-session-auth-and-user-management.md)（§4.7 / §4.5.1） |

---

## 2. 共有クライアント仕様

### 2.1 FirestoreClient (`shared/firestore_client.py`)

#### 責務
Firestore の CRUD を集約する薄いラッパー。外部に Firestore 依存を漏らさない。

#### インターフェース

```python
class FirestoreClient:
    # --- Articles ---
    @staticmethod
    def article_id(url: str) -> str
        """URL の SHA-256 (hex)[:20] を返す。冪等・決定的。"""

    def article_exists(self, article_id: str) -> bool
        """articles/{id} が存在すれば True。"""

    def save_article(self, article: Article) -> None
        """articles/{article.id} を set(merge=True) で保存。id フィールドは除外。"""

    def get_article(self, article_id: str) -> Article | None
        """存在しない場合は None。"""

    def get_recent_articles(self, limit: int = 200) -> list[Article]
        """published_at DESC で最新 limit 件。"""

    # --- UserPrefs ---
    def get_user_prefs(self, user_id: str) -> UserPrefs
        """存在しない場合はデフォルト UserPrefs(user_id, default_difficulty="toeic_900") を返す。"""

    def save_user_prefs(self, prefs: UserPrefs) -> None

    def add_starred_article(self, user_id: str, article_id: str) -> None
        """ArrayUnion で starred_article_ids に追加 (merge=True)。"""

    def add_dismissed_article(self, user_id: str, article_id: str) -> None
        """ArrayUnion で dismissed_article_ids に追加 (merge=True)。"""

    # --- Recommendations ---
    def save_recommendation(self, rec: Recommendation) -> None
        """doc_id = {user_id}_{date}。"""

    def get_recommendation(self, user_id: str, date: str) -> Recommendation | None

    # --- Podcasts ---
    def save_podcast(self, podcast: Podcast) -> None
        """id フィールドは除外して保存。"""

    def get_podcasts_for_user(self, user_id: str, limit: int = 50) -> list[Podcast]
        """created_at DESC。"""

    def podcast_exists_for_article(
        self, user_id: str, article_id: str, difficulty: str
    ) -> bool
        """type="single" で当該 article_id と difficulty の Podcast が存在すれば True。"""
```

#### エラー・エッジケース

| ケース | 挙動 |
|--------|------|
| `article_exists` — ドキュメント不在 | `False` を返す |
| `get_user_prefs` — ドキュメント不在 | `UserPrefs(user_id=user_id, default_difficulty="toeic_900")` を返す |
| `get_article` — ドキュメント不在 | `None` を返す |
| `get_recommendation` — ドキュメント不在 | `None` を返す |
| `save_article` — id フィールド | `model_dump()` から `id` を除外して set。id はドキュメントキーのみ |

---

### 2.2 StorageClient (`shared/storage_client.py`)

#### 責務
Cloud Storage への音声ファイルのアップロードと署名付き URL の生成。

#### インターフェース

```python
class StorageClient:
    def upload_audio(
        self, podcast_id: str, difficulty: str, audio_bytes: bytes
    ) -> str
        """podcasts/{podcast_id}/{difficulty}.mp3 にアップロードし public URL を返す。
        GCS_BUCKET_NAME 環境変数必須。"""

    def get_signed_url(
        self, podcast_id: str, difficulty: str, expiration_seconds: int = 3600
    ) -> str
        """署名付き URL を返す (expiration_seconds 秒間有効)。"""
```

#### エラー・エッジケース

| ケース | 挙動 |
|--------|------|
| `GCS_BUCKET_NAME` 未設定 | `KeyError` を raise（呼び出し元で制御） |
| `audio_bytes` 空バイト列 | GCS にアップロードされるが制御しない（呼び出し元の責任） |

---

### 2.3 GeminiClient (`shared/gemini_client.py`)

#### 責務
Gemini API (テキスト生成 / TTS) のラッパー。

#### インターフェース

```python
class GeminiClient:
    TEXT_MODEL = "gemini-2.5-flash"
    TTS_MODEL  = "gemini-2.5-flash-preview-tts"

    def __init__(self, api_key: str | None = None) -> None
        """api_key 未指定時は GEMINI_API_KEY 環境変数を使用。"""

    def generate_text(self, prompt: str, temperature: float = 0.7) -> str
        """response.text を返す。"""

    def generate_tts(self, text: str, voice: str = "Kore") -> bytes
        """response.candidates[0].content.parts[0].inline_data.data を返す (PCM/WAV bytes)。"""
```

#### エラー・エッジケース

| ケース | 挙動 |
|--------|------|
| `GEMINI_API_KEY` 未設定 | `KeyError` を raise（初期化時） |
| API エラー / タイムアウト | 例外をそのまま伝播（呼び出し元でハンドリング） |

---

## 3. バッチジョブ仕様

### 3.1 RssFetcher (`jobs/rss_fetcher/rss_fetcher.py`)

#### 責務
RSS フィードを feedparser でパースし Article オブジェクトのリストを返す。Firestore アクセスを含まない純粋関数的クラス。

#### インターフェース

```python
class RssFetcher:
    def fetch(self, url: str, source_name: str) -> list[Article]
    def article_id_for(self, url: str) -> str  # SHA-256[:20]
```

#### 入力・出力

- **入力**: RSS フィード URL, ソース名
- **出力**: `Article` オブジェクトのリスト（link が空エントリは除外）

#### エラー・エッジケース

| ケース | 挙動 |
|--------|------|
| エントリに `link` が空/未設定 | そのエントリをスキップ |
| 日付パースエラー | `fetched_at` を `published_at` として使用 |
| feedparser がエラーを返す (bozo=True) | エントリを処理するが bozo エラーは無視（フィールドが取れるものは処理） |
| `content` が RSS に含まれない | `summary` フィールドを使用（空の場合は空文字列） |

#### 実装注意事項
- `entry.link`、`entry.title`、`entry.summary` は **直接属性アクセス** を使用（`entry.get()` は日付フィールドのみ使用）
- 日付パースは `"published"` → `"updated"` → `"created"` の順で試みる

---

### 3.2 ContentExtractor (`jobs/rss_fetcher/content_extractor.py`)

#### 責務
trafilatura を用いて URL から記事本文テキストを抽出する。

#### インターフェース

```python
class ContentExtractor:
    def extract(self, url: str) -> str
```

#### エラー・エッジケース

| ケース | 挙動 |
|--------|------|
| `trafilatura.fetch_url` が `None` を返す | `""` を返す |
| `trafilatura.extract` が `None` を返す | `""` を返す |
| タイムアウト / ネットワークエラー | `trafilatura` 内部で処理され `None` が返る → `""` を返す |

---

### 3.3 RssFetcher ジョブ エントリポイント (`jobs/rss_fetcher/main.py`)

#### 責務
Cloud Run Job のエントリポイント。ユーザーの RSS ソースをフェッチし新規記事を Firestore に保存。

#### 環境変数

| 変数 | 必須 | デフォルト | 説明 |
|------|------|-----------|------|
| `USER_ID` | 任意 | `"default"` | 処理対象ユーザー |
| `GEMINI_API_KEY` | 必須 | — | Gemini API キー |
| `GCS_BUCKET_NAME` | 必須 | — | GCS バケット名 |

#### 動作フロー

1. `USER_ID` の `UserPrefs.rss_sources` を取得
2. 各ソースを `RssFetcher.fetch()` で取得
3. `article_exists()` で重複チェック
4. `content` が 200 文字未満なら `ContentExtractor.extract()` で本文補完
5. `save_article()` で保存

---

### 3.4 Recommender (`jobs/recommendation/recommender.py`)

#### 責務
ユーザーの Star/Dismiss 履歴を Gemini に渡し、候補記事に 0.0〜1.0 のスコアを付ける。

#### インターフェース

```python
class Recommender:
    def __init__(self, gemini_client: GeminiClient | None = None) -> None

    def score_articles(
        self,
        prefs: UserPrefs,
        candidates: list[Article],
        starred_articles: list[Article],
    ) -> list[RecommendedArticle]
```

#### エラー・エッジケース

| ケース | 挙動 |
|--------|------|
| `candidates` が空 | `[]` を返す |
| Gemini API エラー | ログ出力後、全候補に `score=0.5` のフォールバックスコアを返す |
| Gemini がコードブロック付きで JSON を返す | ` ``` ` ブロックを除去して JSON パース |
| `starred_articles` が空 (履歴なし) | 「履歴なし」としてプロンプトを生成。Gemini の推定に委ねる |

---

### 3.5 ScriptGenerator (`jobs/podcast_generator/script_generator.py`)

#### 責務
記事から Podcast スクリプト（日本語イントロ + 英語本編）を Gemini で生成する。

#### インターフェース

```python
@dataclass
class PodcastScript:
    japanese_intro: str
    english_body: str

class ScriptGenerator:
    def __init__(self, gemini_client: GeminiClient | None = None) -> None

    def generate(
        self,
        main_article: Article,
        related_articles: list[Article],
        difficulty: str,   # "toeic_600" | "toeic_900" | "ielts_55" | "ielts_7" | "eiken_2" | "eiken_p1"
        date_str: str,     # "YYYY-MM-DD"
    ) -> PodcastScript
```

#### 難易度指示マッピング

| difficulty | 説明 |
|-----------|------|
| `toeic_600` | 中学〜高校基本語彙。短文・単純な文構造 |
| `toeic_900` | ビジネス基本語彙。複合文あり。標準的英語ニュース番組レベル |
| `ielts_55` | アカデミック語彙。接続詞・節を使った文。NPR ラジオレベル |
| `ielts_7` | 高度な学術・専門語彙。複雑な文構造。ネイティブスピード |
| `eiken_2` | 日常〜社会的話題の語彙。英検2級リスニング問題レベル |
| `eiken_p1` | 時事・専門語彙。論説文レベルの複雑な文構造。英検準1級レベル |

#### エラー・エッジケース

| ケース | 挙動 |
|--------|------|
| Gemini 出力にフォーマット区切り文字がない | `japanese_intro=""` とし `english_body` に全テキストを格納。ログ警告 |
| `related_articles` が空 | プロンプトに「なし」と記載 |
| 未知の `difficulty` 値 | `_DIFFICULTY_INSTRUCTIONS.get(difficulty, difficulty)` でキーをそのまま使用 |

#### 実装注意事項
- プロンプト内に **difficulty キー文字列を含める**（例: `[ielts_7] 高度な学術...`）。テスト `test_generate_uses_difficulty_in_prompt` が `"ielts" in call_args.lower()` を確認するため必須。

---

### 3.6 TtsGenerator (`jobs/podcast_generator/tts_generator.py`)

#### 責務
`PodcastScript` から音声バイト列を生成する。

#### インターフェース

```python
class TtsGenerator:
    def __init__(self, gemini_client: GeminiClient | None = None) -> None

    def generate_audio(self, script: PodcastScript) -> bytes
        """日本語イントロ (voice=Kore) + 英語本編 (voice=Puck) を結合して返す。"""
```

#### 音声仕様

| パート | voice | 言語 |
|--------|-------|------|
| `japanese_intro` | `Kore`（落ち着いた女性） | 日本語 |
| `english_body` | `Puck`（明瞭な男性） | 英語 |

#### エラー・エッジケース

| ケース | 挙動 |
|--------|------|
| Gemini TTS API エラー | 例外をそのまま伝播 |
| `japanese_intro` が空文字列 | 空バイト列を結合（サイズ 0） |

---

## 4. REST API 仕様

### 4.1 FastAPI アプリ (`api/main.py`)

#### 認証ミドルウェア

- **ヘッダー名**: `X-API-Key`
- **検証**: `os.environ["API_KEY"]` と完全一致
- **失敗時**: `HTTP 401 Unauthorized`
- **`GET /health`**: 認証不要（グローバル依存から除外）

#### インターフェース

```python
app = FastAPI(title="Tech News Podcast API", version="0.1.0")

async def verify_api_key(api_key: str = Security(API_KEY_HEADER)) -> str
    """api_key が空・None・不一致なら HTTP 401 を raise。"""
```

---

### 4.2 Feed ルーター (`api/routers/feed.py`)

#### エンドポイント

```
GET /feed
```

**レスポンス** (`FeedResponse`):
```json
{
  "articles": [
    {
      "id": "string",
      "title": "string",
      "url": "string",
      "source": "string",
      "score": 0.95,
      "published_at": "2026-06-10T00:00:00+00:00"
    }
  ],
  "date": "2026-06-10"
}
```

#### エラー・エッジケース

| ケース | 挙動 |
|--------|------|
| 今日の Recommendation が存在しない | `articles=[]` の空フィードを返す（200 OK） |
| Recommendation の article_id に対応する Article が存在しない | そのエントリをスキップ |
| `USER_ID` 未設定 | `"default"` をデフォルト値として使用 |

---

### 4.3 Articles ルーター (`api/routers/articles.py`)

#### エンドポイント

```
POST /articles/{article_id}/star
POST /articles/{article_id}/dismiss
```

**レスポンス** (`ActionResponse`):
```json
{ "status": "starred", "article_id": "abc123" }
{ "status": "dismissed", "article_id": "abc123" }
```

#### エラー・エッジケース

| ケース | レスポンス |
|--------|-----------|
| `article_id` に対応する Article が存在しない | `HTTP 404 Not Found` |
| 既に Star 済みの記事を再度 Star | `ArrayUnion` で重複なし追加（200 OK, 冪等） |

---

### 4.4 Podcasts ルーター (`api/routers/podcasts.py`)

#### エンドポイント

```
GET /podcasts
GET /podcasts/{podcast_id}
```

**GET /podcasts レスポンス** (`PodcastListResponse`):
```json
{
  "podcasts": [
    {
      "id": "uuid",
      "type": "single",
      "article_ids": ["abc123"],
      "difficulty": "toeic_900",
      "audio_url": "https://storage.googleapis.com/...",
      "japanese_intro_text": "...",
      "duration_seconds": 300,
      "created_at": "2026-06-10T00:00:00+00:00"
    }
  ]
}
```

#### エラー・エッジケース

| ケース | レスポンス |
|--------|-----------|
| `podcast_id` に対応する Podcast が存在しない | `HTTP 404 Not Found` |
| ユーザーの Podcast がゼロ件 | `podcasts=[]` の空リストを返す（200 OK） |

---

### 4.5 Settings ルーター (`api/routers/settings.py`)

#### エンドポイント

```
GET  /settings/sources
POST /settings/sources        body: { "name": string, "url": string }
DELETE /settings/sources?url= クエリパラメータ
```

**GET レスポンス** (`RssSourcesResponse`):
```json
{ "sources": [{ "name": "HackerNews", "url": "https://..." }] }
```

おすすめサイト（システム提供）参照・オンボーディングも本ルーターが担う:

```
GET  /settings/featured-sources    # システム提供のおすすめサイト一覧（order 昇順）
GET  /settings/onboarding          # 初回オンボーディング完了状態
POST /settings/onboarding/complete # 初回オンボーディング完了を記録
```

**GET /settings/featured-sources レスポンス** (`FeaturedSitesResponse`):
```json
{ "sites": [
  { "id": "the-verge", "name": "The Verge", "url": "https://www.theverge.com/rss/index.xml",
    "thumbnail_url": "https://www.theverge.com/favicon.ico", "description": "テクノロジー全般" }
] }
```
- `featuredSites` コレクション（グローバル・ユーザー横断）を `order` 昇順で返す。表示順は配列順で表現し、`order` はレスポンスに含めない。

**GET /settings/onboarding ・ POST /settings/onboarding/complete レスポンス** (`OnboardingStatusResponse`):
```json
{ "onboarding_completed": false }
```
- 状態は `UserPrefs.onboarding_completed`（単一ユーザー前提のためデプロイ単位で1値）に永続化する。
- `complete` は `get_user_prefs → model_copy(update={onboarding_completed: True}) → save_user_prefs` の全置換更新（`add_source` と同流儀。required な `default_difficulty` も保持される）。

#### エラー・エッジケース

| ケース | レスポンス |
|--------|-----------|
| POST /settings/sources — 既存 URL と重複 | `HTTP 409 Conflict` |
| DELETE /settings/sources — 存在しない URL | `HTTP 404 Not Found` |
| GET /settings/sources — RSS ソースが0件 | `sources=[]` の空配列（200 OK） |
| GET /settings/featured-sources — 0件 | `sites=[]` の空配列（200 OK） |
| GET /settings/onboarding — 既存値なし | `onboarding_completed=false`（default） |

---

### 4.5.1 Admin ルーター (`api/routers/admin.py`)

管理操作を集約する。**おすすめサイト CRUD は共有 `X-API-Key`**、**ユーザー管理 CRUD は `require_admin`（admin ロール）**で保護する（認証方針の差は下記「認証」と [ADR-013](../adr/013-session-auth-and-user-management.md)）。

#### おすすめサイト（`featuredSites`）

運用者専用で、通常クライアント（Web/iOS）からは利用しない。

```
GET    /admin/featured-sites               # 一覧
POST   /admin/featured-sites               # 作成（id は name から slug 化）
PUT    /admin/featured-sites/{site_id}     # 全置換更新
DELETE /admin/featured-sites/{site_id}     # 削除
```

#### ユーザー管理（`users`・admin ロール限定）

```
GET    /admin/users                # ユーザー一覧
POST   /admin/users                # 作成（username / password / role / display_name）
PATCH  /admin/users/{username}     # ロール変更・PW リセット・表示名変更（指定フィールドのみ）
DELETE /admin/users/{username}     # 削除
```

- すべて `require_admin` で保護（非 admin は `403`）。
- **最後の admin 保護**: `_admin_count(db) <= 1` の状態で唯一の admin を `user` へ降格、または削除しようとすると `HTTP 409`。
- **セッション失効**: ロール降格（admin→user）・パスワードリセット（`new_password` 指定）・ユーザー削除のいずれでも、対象ユーザーの全セッションを `delete_sessions_for_user` で即時失効し、旧権限・旧資格情報での継続アクセスを断つ。
- **自己ロックアウト防止**: Web/iOS の管理画面は自分自身のロール変更・削除ボタンを出さない（サーバ側は上記 409 で二重防御）。

**POST/PUT リクエスト** (`FeaturedSiteRequest`):
```json
{ "name": "Wired (Technology)", "url": "https://www.wired.com/feed/...",
  "thumbnail_url": "https://www.wired.com/favicon.ico", "description": "テクノロジー", "order": 3 }
```
- `url` / `thumbnail_url` は `HttpUrl`（SSRF 軽減、`RssSourceRequest` と同方針）。
- doc id は `slugify(name)`（例: `"Wired (Technology)"` → `wired-technology`）。

#### 認証

- **おすすめサイト CRUD**: 専用ロールを持たず共有 `X-API-Key`（`verify_api_key`）で保護（ADR-012 (C)）。運用者専用の位置づけ。
- **ユーザー管理 CRUD**: `require_admin`（ログインセッションの `role == "admin"`）で保護（ADR-013）。マルチユーザー化に伴い、ユーザー発行・停止は admin に限定する。

#### エラー・エッジケース

| ケース | レスポンス |
|--------|-----------|
| featured-sites POST — 既存 slug と重複 | `HTTP 409 Conflict` |
| featured-sites PUT/DELETE — 存在しない site_id | `HTTP 404 Not Found` |
| featured-sites — `X-API-Key` 不正/欠落 | `HTTP 401 Unauthorized` |
| users — 未認証（セッション無効） | `HTTP 401 Unauthorized` |
| users — 非 admin | `HTTP 403 Forbidden` |
| users POST — 既存 username と重複 | `HTTP 409 Conflict` |
| users PATCH/DELETE — 存在しない username | `HTTP 404 Not Found` |
| users — 最後の admin の降格/削除 | `HTTP 409 Conflict` |

#### 初期データ投入

- `backend/scripts/seed_featured_sites.py`: `FirestoreClient.save_featured_site` を直接呼び、デフォルト7サイト（The Verge / TechCrunch / Engadget / Wired (Technology) / Mashable / HackerNews / VentureBeat）を冪等投入。
- `backend/scripts/seed_users.py`: 環境変数（`INITIAL_ADMIN_USERNAME`/`INITIAL_ADMIN_PASS`・`INITIAL_USER_USERNAME`/`INITIAL_USER_PASSWORD`）から初期 admin / user を冪等投入（既存ユーザーは上書きしない）。**初回デプロイ後に既定パスワードを変更すること**。

---

### 4.6 Pydantic スキーマ (`api/schemas.py`)

```python
class ArticleResponse(BaseModel):
    id: str; title: str; url: str; source: str
    score: float; published_at: str  # ISO 8601

class FeedResponse(BaseModel):
    articles: list[ArticleResponse]; date: str

class PodcastResponse(BaseModel):
    id: str; type: str; article_ids: list[str]; difficulty: str
    audio_url: str; japanese_intro_text: str
    duration_seconds: int; created_at: str

class PodcastListResponse(BaseModel):
    podcasts: list[PodcastResponse]

class RssSourceRequest(BaseModel):
    name: str; url: str

class RssSourcesResponse(BaseModel):
    sources: list[dict]

class ActionResponse(BaseModel):
    status: str; article_id: str

# おすすめサイト / オンボーディング
class FeaturedSiteResponse(BaseModel):
    id: str; name: str; url: str
    thumbnail_url: str | None = None; description: str | None = None

class FeaturedSitesResponse(BaseModel):
    sites: list[FeaturedSiteResponse]

class FeaturedSiteRequest(BaseModel):           # 管理用（admin）
    name: str; url: HttpUrl
    thumbnail_url: HttpUrl | None = None
    description: str | None = None; order: int = 0

class OnboardingStatusResponse(BaseModel):
    onboarding_completed: bool
```

**データモデル追記** (`shared/models.py`):
```python
class FeaturedSite(BaseModel):                  # Firestore: featuredSites/{id}
    id: str; name: str; url: str
    thumbnail_url: str | None = None
    description: str | None = None; order: int = 0

class UserPrefs(BaseModel):
    ...
    onboarding_completed: bool = False          # 既存ドキュメントは default で後方互換

# 認証（ADR-013）
UserRole = Literal["admin", "user"]

class User(BaseModel):                          # Firestore: users/{username}
    username: str; user_id: str; password_hash: str
    role: UserRole; display_name: str
    created_at: datetime; updated_at: datetime

class Session(BaseModel):                       # Firestore: sessions/{session_id}
    session_id: str                             # 発行トークンの SHA-256 ハッシュ
    user_id: str; username: str; role: UserRole
    created_at: datetime; expires_at: datetime
```

---

### 4.7 認証ルーター・セッション (`api/routers/auth.py` / `api/dependencies.py`)

ADR-013 のセッションベース認証。利用者は username + パスワードでログインし、以降は不透明トークンで認証する。

#### エンドポイント

```
POST  /auth/login        body: { "username": str, "password": str }   # 認証不要
POST  /auth/logout                                                    # 認証不要・冪等
GET   /auth/me                                                        # 要・get_current_user
PATCH /auth/me           body: { "display_name": str }                # 要・get_current_user
POST  /auth/password     body: { "current_password", "new_password" } # 要・get_current_user
```

- **login**: 資格情報を検証し、`generate_session_token()` で生トークンを発行 → `hash_token()`（SHA-256）した値を `Session` として保存。Web へは `Set-Cookie: nl_session=...`（httpOnly / `SESSION_COOKIE_SECURE` で Secure 制御）、iOS へはボディの `token` で返す（`LoginResponse { token, user }`）。
- **login 失敗**: ユーザー存在の有無を伏せた汎用メッセージで `HTTP 401`（`Invalid username or password`）。
- **logout**: 現在のセッションを削除し Cookie を失効。トークンが無くても 200（冪等）。
- **password**: `current_password` を検証してから更新。

#### セキュリティヘルパー (`shared/security.py`)

| 関数 | 役割 |
|---|---|
| `hash_password` / `verify_password` | bcrypt によるハッシュ化・検証（UTF-8 72 バイト超は切り詰め） |
| `generate_session_token` | `secrets.token_urlsafe(32)` による推測不能トークン |
| `hash_token` | トークンの SHA-256 ハッシュ（DB 保存用。生トークンは保存しない） |

#### 認証依存 (`api/dependencies.py`)

- **トークン抽出順序**: ① `Authorization: Bearer <token>`（iOS）→ ② Cookie `nl_session`（Web、`SESSION_COOKIE_NAME`）。
- `get_current_user(request, db) -> Session`: 生トークン → SHA-256 → Firestore 照合。`expires_at` 超過は遅延削除し未認証扱い。無効は `HTTP 401`。
- `get_user_id(current) -> str`: ログイン中の `user_id` を返す（**従来の環境変数 `USER_ID` 固定からセッション由来へ変更**。ADR-007 の単一ユーザー前提を更新）。
- `require_admin(current) -> Session`: `role != "admin"` は `HTTP 403`。
- エラーメッセージに内部情報・トークンを含めない。

#### 環境変数（認証関連・新規）

| 変数 | 既定 | 説明 |
|---|---|---|
| `SESSION_TTL_HOURS` | 168（7日） | セッション有効期間 |
| `SESSION_COOKIE_SECURE` | true | Cookie の Secure 属性（ローカル開発は false） |
| `INITIAL_ADMIN_USERNAME` / `INITIAL_ADMIN_PASS` | なし | 初期 admin（`seed_users.py`） |
| `INITIAL_USER_USERNAME` / `INITIAL_USER_PASSWORD` | なし | 初期 user（`seed_users.py`） |

---

## 5. Docker 仕様 (`Task 13`)

### 5.1 Dockerfile.jobs

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY shared/ ./shared/
COPY jobs/ ./jobs/
CMD python -m ${JOB_MODULE}
```

### 5.2 Dockerfile.api

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY shared/ ./shared/
COPY api/ ./api/
EXPOSE 8080
CMD ["uvicorn", "api.main:app", "--host", "0.0.0.0", "--port", "8080"]
```

### 5.3 .dockerignore

```
.venv/
__pycache__/
*.pyc
*.pyo
tests/
.pytest_cache/
```

---

## 6. テスト方針

### 6.1 モック戦略

| 依存 | モック方法 |
|------|-----------|
| `FirestoreClient` | `conftest.py` の `mock_firestore_db` fixture を使用。`patch("shared.firestore_client.firestore.Client")` |
| `GeminiClient` | `MagicMock()` を DI パラメータとして渡す |
| `feedparser.parse` | `patch("jobs.rss_fetcher.rss_fetcher.feedparser.parse")` |
| `trafilatura` | `patch("jobs.rss_fetcher.content_extractor.trafilatura.fetch_url/extract")` |
| FastAPI テストクライアント | `TestClient(app)` + `patch("api.routers.*.FirestoreClient")` |

### 6.2 conftest.py の位置

`backend/tests/conftest.py` に一元配置（`pyproject.toml` の `testpaths = ["tests"]` と一致）。

```python
# backend/tests/conftest.py
import pytest
from unittest.mock import MagicMock, patch

@pytest.fixture
def mock_firestore_db():
    with patch("shared.firestore_client.firestore.Client") as mock_client_class:
        mock_db = MagicMock()
        mock_client_class.return_value = mock_db
        yield mock_db
```

### 6.3 APIテストフィクスチャの注意事項

全 API テストのフィクスチャは `return` ではなく `yield` を使用すること。`return` を使うと `patch.dict` コンテキストが終了して環境変数パッチが外れる。

```python
@pytest.fixture
def client():
    with patch.dict("os.environ", {"API_KEY": "test-key", "USER_ID": "user1"}):
        import importlib
        import api.main as m
        importlib.reload(m)
        yield TestClient(m.app)  # yield — patch.dict が test 実行中も有効
```

---

## 7. 制約・制限事項

- **Task 13**: Dockerfile の作成 (Step 1〜3) のみ。GCP デプロイ (Step 4〜7) は実行しない
- **GCS バケット名**: `news-listen-20260610-podcasts`（環境変数 `GCS_BUCKET_NAME` で注入）
- **サービスアカウント**: `news-listen-sa@news-listen-20260610.iam.gserviceaccount.com`
- **シークレット名**: `news-listen-api-key` (API キー), `news-listen-gemini-key` (Gemini API キー)
- **Difficulty フィールド**: Enum 廃止。plain string として扱う（`"toeic_900"` 等）
