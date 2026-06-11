# Web フロントエンド仕様書（spec）

**作成日:** 2026-06-10
**スコープ:** `web/` 配下に新規実装する Next.js Web フロントエンド
**正の優先順位:** `backend/api/`（実コード）> 本仕様書 > `docs/plan/2026-06-10-web-frontend.md` > PRD（`docs/prd/2026-05-31-news-listen.md`）> `docs/design/web-design.html` > order.md
**対応実装計画:** `docs/plan/2026-06-10-web-frontend.md`（T1〜T17）

> 本仕様の API 契約は `backend/api/main.py`・`backend/api/routers/*.py`・`backend/api/schemas.py` を 2026-06-10 時点で直接読了して転記したものである。`docs/design/web-design.html` にはバックエンドに存在しないエンドポイント（`/subscriptions/:id`、`PATCH /podcasts/:id/position`、`GET/PUT /settings`、Podcast `status`）への言及があるが、**すべて本仕様を正とする**。

---

## 1. 目的

既存バックエンド API（FastAPI / Cloud Run）を利用し、以下を提供する Web アプリを実装する:

- Feed 閲覧と記事の Star / Dismiss
- Podcast（生成済み英語音声）の一覧・詳細・再生（速度変更・シーク・音量・位置復元）
- RSS 購読ソースの一覧・追加・削除
- API 接続設定（ベース URL・API キー）とデフォルト再生速度の管理

## 2. 技術スタックとアーキテクチャ決定

| 項目 | 決定 | 根拠 |
|---|---|---|
| フレームワーク | Next.js 15 / App Router / TypeScript | 設計書（`web-design.html:82,84,95`）で確定済み。PRD にフロントエンド技術選定の記載はない（grep 0 件を確認済み）。Pages Router は採用しない |
| スタイリング | Tailwind CSS v4 + CSS 変数トークン | 設計書（`web-design.html:86,97`） |
| 状態管理 | React Context + useReducer（ライブラリ不使用） | 規模に対して十分。§7 参照 |
| テスト | Vitest + React Testing Library + jsdom | TDD 必須（agent-rules/11） |
| 通信経路 | ブラウザ → Next.js Route Handler（BFF プロキシ）→ バックエンド | バックエンドに CORS ミドルウェアが**ない**（`backend/api/` 全域に grep ヒット 0 件を確認済み）ため直接 fetch は不可 |
| レンダリング | データ取得を伴うページは全て `'use client'` | API キーが localStorage にあるため Server Components でのデータ取得は不可能 |
| デプロイ形態 | `output: 'standalone'` + Docker (node:22-alpine) | Cloud Run 用。デプロイ実行はスコープ外 |

**採らなかった代替案**（検討済み・実装禁止）:
- バックエンドへの CORSMiddleware 追加 → backend 変更が必要で「1 ブランチ = 1 目的」に反する。Phase 2
- Next.js rewrites → 転送先がビルド時固定になり、実行時ユーザー設定（localStorage）と矛盾
- `NEXT_PUBLIC_API_BASE_URL` 環境変数（order.md タスク 4 記載）→ ビルド時固定のため**採用しない**

## 3. バックエンド API 契約（実コードからの転記 — 検証済み）

全エンドポイント（`/health` 以外）は `X-API-Key` ヘッダー必須。キー不一致・欠落は `401 {"detail": "Invalid or missing API key"}`（`backend/api/main.py:22-30`）。

| エンドポイント | 正常レスポンス | エラー |
|---|---|---|
| `GET /health` | `{"status": "ok"}` | なし（認証不要） |
| `GET /feed` | `{"articles": ArticleResponse[], "date": "YYYY-MM-DD"}`。未生成時は `articles: []` で 200。最大 50 件 | 401 |
| `POST /articles/{id}/star` | `{"status": "starred", "article_id": id}`。冪等 | 404 `Article not found` / 401 |
| `POST /articles/{id}/dismiss` | `{"status": "dismissed", "article_id": id}`。冪等。**サーバーは当日 feed から除去しない** | 404 `Article not found` / 401 |
| `GET /podcasts` | `{"podcasts": PodcastResponse[]}`。`audio_url` は**有効期限 1 時間の署名付き GCS URL** | 401 |
| `GET /podcasts/{id}` | `PodcastResponse` 単体 | 404 `Podcast not found` / 401 |
| `GET /settings/sources` | `{"sources": [{"name", "url"}]}` | 401 |
| `POST /settings/sources` | 更新後の**全 sources**（再 GET 不要） | 409 `Source URL already exists` / 422（Pydantic HttpUrl 形式エラー）/ 401 |
| `DELETE /settings/sources?url={url}` | 更新後の**全 sources** | 404 `Source not found` / 401 |

**存在しないもの（実装・呼び出し禁止）:** `PodcastResponse.status`、`playback_position_seconds`、`PATCH /podcasts/:id/position`、`GET/PUT /settings`、`/subscriptions/*`、ソースの `id`・`enabled`。

## 4. 型定義（`web/types/index.ts`）

フィールド名は API レスポンスの **snake_case のまま**使う（変換層を挟まない。WHY: バックエンド変更時の追従コストを下げる）。

```typescript
type Difficulty = 'toeic_600' | 'toeic_900' | 'ielts_55' | 'ielts_7' | 'eiken_2' | 'eiken_p1';
type Article = { id: string; title: string; url: string; source: string;
                 score: number; published_at: string };
type Podcast = { id: string; type: 'single' | 'digest'; article_ids: string[];
                 difficulty: Difficulty; audio_url: string;  // 常に string（null 不可）
                 japanese_intro_text: string; duration_seconds: number; created_at: string };
type RssSource = { name: string; url: string };
type FeedResponse = { articles: Article[]; date: string };
type PodcastListResponse = { podcasts: Podcast[] };
type RssSourcesResponse = { sources: RssSource[] };
type ActionResponse = { status: string; article_id: string };
```

## 5. API クライアント（`web/lib/api.ts`）

### 5.1 公開インターフェース

`createApiClient(config: { baseUrl: string; apiKey: string })` で生成。**モジュールロード時に localStorage を読まない**（SSR 安全性・テスト容易性）。公開関数:

`getFeed()` / `starArticle(id)` / `dismissArticle(id)` / `getPodcasts()` / `getPodcast(id)` / `getSources()` / `addSource(name, url)` / `deleteSource(url)` / `checkHealth()`

すべて同一オリジンの `/api/backend/{path}` へ fetch し、ヘッダーに `X-API-Key`（ユーザー入力キー）と `X-Backend-Base-Url`（ユーザー入力ベース URL）を付与する。`deleteSource` は `url` を `encodeURIComponent` してクエリに載せる（`&`・日本語を含む URL でも壊れないこと）。

### 5.2 エラー正規化（異常系の唯一の契約）

```typescript
class ApiError extends Error { status: number; detail: string }
```

| 入力 | 振る舞い |
|---|---|
| 非 2xx + `{"detail": "..."}` ボディ | `ApiError(status, detail)` を throw |
| 非 2xx + JSON でない / `detail` 欠落ボディ | `ApiError(status, 'Unknown error')` を throw |
| fetch reject（ネットワーク断） | `ApiError(0, ...)` に正規化して throw |
| 2xx | パース済み JSON を型付きで返す |

**禁止:** API キー・署名付き URL の `console.log` 出力（agent-rules/12）。

## 6. BFF プロキシ（`web/app/api/backend/[...path]/route.ts`）

| 項目 | 仕様 |
|---|---|
| 入力 | 任意のパス・クエリ・ボディ・メソッド（GET/POST/PATCH/DELETE を export） |
| 転送 | `X-Backend-Base-Url` の値 + path + クエリ文字列へ転送。`X-API-Key`・`Content-Type`・ボディ・メソッドを引き継ぐ |
| 出力 | バックエンドのステータスコードとボディを**加工せず**素通し（エラー解釈はクライアント `ApiError` の責務） |
| 異常系: ヘッダー欠落 | `X-Backend-Base-Url` がない → **400** |
| 異常系: 不正スキーム | `http://`・`https://` 以外（`ftp://`、`file://`、相対等）→ **400**（SSRF 緩和） |
| 異常系: バックエンド到達不能 | fetch reject → 502 を返す（ボディは `{"detail": "Bad Gateway"}` 相当で可） |

## 7. グローバル状態（`web/contexts/AppContext.tsx`）

| 状態 | 内容 | 永続化 |
|---|---|---|
| API 設定 | `baseUrl` / `apiKey` / `isConfigured` | localStorage |
| デフォルト再生速度 | 8 段階のいずれか | localStorage |
| 再生状態 | `currentPodcast` / `isPlaying` / `currentTime` / `duration` / `playbackSpeed` | なし（位置のみ §8） |

- reducer の action: `SET_PODCAST` / `PLAY` / `PAUSE` / `SET_TIME` / `SET_SPEED` / `CONFIGURE` / `RESTORE`
- `useApp()` フックで公開。**Provider 外で呼ぶと throw**
- localStorage 復元は `useEffect` 内（マウント後）で行い、復元前は `isConfigured: false` のまま（hydration 安全）
- 一覧データ（articles / podcasts / sources）・isLoading・error は**各ページのローカル state**。Context に入れない
- order.md タスク 2 の「アクティブ画面の Context 管理」は**実装しない**。画面状態は URL（App Router）が正であり二重管理は状態不整合の温床

## 8. ローカル永続化（`web/hooks/useLocalStorage.ts` + `web/lib/config.ts`）

キー定数は `lib/config.ts` に集約: `api_base_url` / `api_key` / `default_playback_speed` / `player_volume` / `podcast_position:{podcastId}`（プレフィックス定数 + キー生成関数）。

| 異常系 | 振る舞い |
|---|---|
| `window` 不在（SSR） | 例外を出さず初期値を返す |
| 保存値が JSON 不正 | 初期値へフォールバック（例外を出さない） |
| ページコンポーネントから直接 `localStorage` を触る | **禁止**。必ずこのフック経由 |

## 9. 音声再生（`web/hooks/useAudioPlayer.ts`）

- HTML5 `Audio` を直接使用。再生ロジックは UI から完全分離（フック単体でテスト可能）
- 速度 8 段階 `[0.5, 0.75, 1.0, 1.25, 1.5, 1.75, 2.0, 2.5]` を定数として export

| イベント / 操作 | 仕様 |
|---|---|
| `load(url, resumePosition)` | `currentTime` を復元。**`resumePosition >= duration` の場合は 0 から再生**（音声差し替え等での範囲外保存値対策） |
| `timeupdate` | 現在位置を更新し、**10 秒間隔のスロットル**で `podcast_position:{id}` へ保存 |
| `ended` | `isPlaying: false`、保存位置を 0 にリセット |
| `error`（Audio 要素のエラーイベント） | `isPlaying: false` にし、エラーを呼び出し元へ通知（UI はトースト「音声を再生できません」を表示） |
| 別エピソードの `load` / アンマウント | 旧 Audio を pause し `src` を解放（リーク防止） |
| シーク | -15 秒 / +30 秒ボタン + シークバー |
| 音量（order.md タスク 6 の明示要求） | `setVolume(v)`: v を **[0, 1] にクランプ**して `Audio.volume` へ反映し、`player_volume` キー（§8）へ保存。`load` 時に保存値を適用（未保存・不正値は `1.0` へフォールバック） |

音量はフック内のローカル状態とし、AppContext（§7）には**置かない**（WHY: 音量を参照する UI は AudioPlayerBar のみで、グローバル状態に載せると再生バー以外の再レンダリングを誘発するうえ二重管理になる。永続化は §8 のフック経由で完結する）。

**署名付き URL 失効対策（重要）:** 一覧取得時の `audio_url` は再生に使わない。再生ボタン押下時に必ず `getPodcast(id)` を呼び直し、新鮮な URL を取得してから `Audio` にセットする。

## 10. 画面仕様

ルーティング: `/`（ゲート）、`/feed`、`/podcast`、`/podcast/[id]`、`/subscriptions`、`/settings` の 4 画面 + 詳細 + エントリー。order.md の「3 画面」は誤り（Subscriptions を含む）。NavigationBar は 4 リンク（Feed / Podcast / Subscriptions / Settings）で現在パスに `aria-current="page"`。モバイル（375px）では下部タブ化。

### 10.1 `/`（エントリーゲート + SetupModal）

| 状態 | 挙動 |
|---|---|
| 設定復元前 | スケルトン表示（チラつき防止） |
| 設定済み | `/feed` へ `router.replace` |
| 未設定 | SetupModal を表示。閉じて他画面を使うことはできない |

SetupModal バリデーション（クライアント側・インラインエラー表示・保存不可）:
- baseUrl: 空 → 不可。`https://` で始まらない → 不可
- apiKey: 空 → 不可。入力欄は `type="password"`
- 「接続テスト」ボタン: `/api/backend/health` を叩き成否をインライン表示（`/health` は認証不要のため**疎通確認**。キー検証は `getFeed()` の 401 有無で行う）

### 10.2 `/feed`

| 状態 | 挙動 |
|---|---|
| ローディング | SkeletonCard |
| 正常（1 件以上） | ArticleCard 一覧（score 降順は API 保証）。スタッガードリビール演出 |
| 空（`articles: []`） | 「まだ記事がありません。バッチは毎日 06:00 に実行されます。」 |
| 取得エラー | エラーメッセージ + 再試行ボタン |

ArticleCard（制御コンポーネント。props: `{ article, onStar, onDismiss, busy }`。API 呼び出しはページの責務）:
- タイトル＝外部リンク（`target="_blank"` + `rel="noopener noreferrer"`）、ソース、公開日、スコアバー（width がスコア比例、`aria-valuenow`）
- 操作中（busy）はボタン disabled（二重送信防止）

| 操作 | 結果 |
|---|---|
| Star 成功 | トースト「Star しました。Podcast は次回バッチ実行時に生成されます」。**カードは残し** ★ を点灯 |
| Dismiss 成功 | カードを即時除去（クライアント責務） |
| Star/Dismiss 404 | トースト「記事が見つかりません」+ カード除去 |
| 401 | トースト「API キーが正しくありません。設定を確認してください」+ `/settings` 誘導 |
| ネットワーク断（status 0） | トースト「サーバーに接続できません」 |

楽観更新は**しない**（API 成功後に state 更新。個人利用でロールバック実装に見合わない）。リフレッシュボタンで再取得。

### 10.3 `/podcast`（一覧）と `/podcast/[id]`（詳細）

一覧:
| 状態 | 挙動 |
|---|---|
| ローディング | Skeleton |
| 正常 | PodcastCard 一覧（`created_at` 降順は API 保証） |
| 空 | 「Podcast がまだありません。Feed で記事を Star すると次回バッチで生成されます。」 |
| 再生ボタン | `getPodcast(id)` を呼び直し → AppContext へ `SET_PODCAST` + 再生開始 + `podcast_position:{id}` から復元位置を渡す |
| リフレッシュボタン | 再取得（status ポーリングの代替。ポーリング・StatusBadge は**実装しない**） |

PodcastCard: イントロ先頭 80 文字・DifficultyBadge・`formatDuration`・生成日・保存済み位置があれば「続きから MM:SS」。カード本体クリックで `/podcast/:id` へ。

詳細（`/podcast/[id]`）:
| 状態 | 挙動 |
|---|---|
| 正常 | 日本語イントロ**全文**・難易度・時間・生成日・元記事 ID 表示 + 再生ボタン（一覧と同フロー） |
| 404 | 「エピソードが見つかりません」+ 一覧へ戻るリンク |

AudioPlayerBar（全画面共通・画面下部固定・`app/layout.tsx` 組込み）:
- `currentPodcast` が null なら非表示。null でなければ: イントロ先頭 50 文字 + DifficultyBadge + 再生/一時停止 + -15s/+30s + シークバー（`input[type=range]`、`aria-label`）+ **音量スライダー（`input[type=range]`・0〜100・`aria-label="音量"`。§9 の `setVolume` に接続。order.md タスク 6 の明示要求）** + 速度セレクタ（8 段階、初期値はデフォルト速度）+ `formatDuration` 形式の時間表示
- ページ遷移しても再生継続（レイアウト常駐）。`<main>` 下端にバー高さ分の padding

### 10.4 `/subscriptions`

| 状態 | 挙動 |
|---|---|
| 正常 | SubscriptionRow 一覧 + AddSubscriptionForm |
| 空 | 「購読ソースがありません」+ 追加フォームへの誘導 |
| 追加・削除成功 | レスポンス（更新後全件）で state を**置き換える**（再 GET しない） |

AddSubscriptionForm の異常系:
| 入力 / 結果 | 挙動 |
|---|---|
| URL が `http(s)://` 始まりでない | クライアント側インラインエラー、送信しない |
| 409 | 「この URL は登録済みです」をインライン表示。**入力値は保持** |
| 422 | 「URL の形式が正しくありません」 |
| 送信中 | 入力・ボタン disabled |
| 送信成功 | 入力クリア |

削除: 削除ボタン → ConfirmDialog（Escape で閉じる）→ 確認後 `deleteSource(url)`。404 → トースト「対象が見つかりません」+ 一覧再取得。enabled トグル・行 ID は**実装しない**（API に存在しない）。

### 10.5 `/settings`

- 現在の baseUrl 表示。apiKey は**値を出さず**「設定済み」表示 + 再入力欄
- 保存は AppContext `configure()` 経由（localStorage 直接アクセス禁止）
- デフォルト再生速度セレクタ（8 段階定数を import）→ localStorage 保存 + AppContext 反映
- 接続テストボタン（SetupModal とロジック共通化）
- 難易度設定 UI は**置かない**。「Podcast の難易度はサーバー側設定で管理されています」の説明文を表示

### 10.6 汎用 UI 部品

| 部品 | 仕様（異常系含む） |
|---|---|
| Toast | `useToast()`（Context）で全画面から呼ぶ。3 秒で自動消滅。success は `role="status"`、error は `role="alert"` |
| DifficultyBadge | 6 難易度にラベル（例: `toeic_900` → 「TOEIC 900」）と色。**未知の文字列でも例外を出さず生値を表示** |
| SkeletonCard | ローディング表示 |
| ConfirmDialog | 開閉・確認/キャンセルコールバック・Escape で閉じる |
| `lib/format.ts` | `formatDuration(300)` → `"5:00"`。0 秒・3600 秒以上も正しく整形。`formatDate(ISO)` → 「6/10 09:00」。**不正 ISO 文字列で例外を出さない** |

## 11. デザイン仕様（agent-rules/15 準拠）

- トーン: ダーク・エディトリアル。CSS 変数トークン（web-design.html の値をそのまま採用）:
  `--bg: #0f1117` / `--surface: #1a1d27` / `--border: #2a2d3e` / `--text: #e2e4ed` / `--muted: #8b8fa8` / `--accent: #7c6af7` / `--accent2: #4fc3f7` / `--green: #4caf82` / `--orange: #f0a050` / `--red: #e05c5c`
- フォント（**禁止: Inter / Roboto / Arial / system-ui / Space Grotesk** — agent-rules/15-frontend-design.md:24,42 で確認済み）: 見出し欧文 = Bricolage Grotesque、和文本文 = Zen Kaku Gothic New、数値・メタ = IBM Plex Mono（すべて next/font/google）
- モーション: ロード時スタッガードリビール（CSS のみ）+ Star/Dismiss 時のカード除去トランジション。常時動くアニメーション禁止

## 12. テスト戦略（TDD 必須）

- 各タスクで**先に失敗するテスト**を書く（Red → Green → Refactor）。Red 観点は計画書 T1〜T17 に明記済み
- fetch は `vi.stubGlobal('fetch', vi.fn())`。`Audio` は jsdom に実装がないため `tests/helpers/mockAudio.ts` にモッククラス（play/pause/currentTime/playbackRate/イベント発火ヘルパー）
- テスト配置: `web/tests/` 配下にソースと同階層構造
- 必須観点: エラーハンドリング・境界値（score 0/1.0、duration 0/3600+、音量の範囲外値 <0 / >1 のクランプ、未知 difficulty、JSON 不正、`&`・日本語入り URL）・インターフェース契約（必須フィールド検証）
- `npm run test` 全件 10 秒以内

## 13. セキュリティ要件

1. BFF プロキシ: `X-Backend-Base-Url` の `http(s)://` スキーム検証（SSRF 緩和）
2. API キー・署名付き URL をログ・コミットに含めない
3. apiKey 入力は `type="password"`、設定画面で値を表示しない
4. 外部リンクは `rel="noopener noreferrer"`

## 14. 受け入れ基準

1. `cd web && npm run test` — 全テストパス、10 秒以内
2. `npm run build` — 型・lint エラーなし
3. 手動シナリオ: 初回 SetupModal → 接続テスト → /feed → Star（トースト・★点灯）/ Dismiss（カード消失）→ /podcast 再生 → 速度変更・音量変更・シーク → 画面遷移で再生継続 → リロード後に前回位置から再開 → /subscriptions 追加（409 文言）・削除（ダイアログ）→ 誤キーで 401 トースト
4. `docker build` 成功・コンテナ起動確認
5. API キー・署名付き URL がコンソールログに出ない

## 15. スコープ外（Phase 2 — 着手禁止）

バックエンド変更全般（CORS 追加・status 公開・`PATCH /podcasts/:id/position`・`GET/PUT /settings`）、status ポーリング表示、再生位置サーバー同期、難易度設定 UI、E2E（Playwright）、Web Push、オフライン再生、Cloud Run 実デプロイ。必要を感じても実装せず計画更新を提案すること。
