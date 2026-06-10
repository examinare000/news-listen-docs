# Web フロントエンド実装計画書（Coder 向け実装指示書）

> **For agentic workers:** 本計画はタスク単位で順番に実装すること。各タスクは TDD（Red → Green → Refactor）で進め、ステップはチェックボックス（`- [ ]`）で進捗管理する。具体的なコードは本書に含めない。本書の「契約」（型・レスポンス形・ファイルパス・テスト観点）から逸脱しないこと。

**作成日:** 2026-06-10
**対象ブランチ:** `dev/web`
**Goal:** 既存バックエンド API（Cloud Run 上の FastAPI）を利用する Web フロントエンド（Next.js）を `web/` 配下に新規実装し、Feed 閲覧・Star/Dismiss・Podcast 再生・RSS 購読管理・設定を提供する。

**Architecture:** Next.js 15 (App Router) + TypeScript + Tailwind CSS v4。全ページをクライアントコンポーネント中心で構成し、ブラウザ → Next.js Route Handler（BFF プロキシ）→ バックエンド API という経路で通信する（CORS 回避のため。§3.1 参照）。音声再生は HTML5 Audio API を直接使用。

**Tech Stack:** Next.js 15 / TypeScript / Tailwind CSS v4 / Vitest + React Testing Library (jsdom) / Docker (node:22-alpine) / Cloud Run

---

## 0. 前提・参照ドキュメント

| ドキュメント | 用途 |
|---|---|
| `docs/design/web-design.html` | 画面・コンポーネント設計の原本（ただし §2 の差分決定が優先） |
| `docs/prd/2026-05-31-audio-news.md` | 機能要件・優先度の正本 |
| `spec.md` | バックエンド API 実装仕様（実装済み・90 テストパス） |
| `backend/api/` | **API 契約の最終的な正**。本書 §4 はここから転記した |
| `agent-rules/11-testing-strategy.md` | TDD 必須。テスト先行作成 |
| `agent-rules/15-frontend-design.md` | デザインガイドライン（禁止フォント等） |
| `agent-rules/10-git-strategy.md` | アトミックコミット・コミットメッセージ規約 |

**重要:** `docs/design/web-design.html` には**バックエンドに存在しないエンドポイント**への言及がある。実装時は必ず本書 §2・§4 を正とすること。

---

## 1. スコープ

### 含む（本計画の成果物）

- `web/` 配下の Next.js アプリ一式（画面 5 種 + 共通コンポーネント + BFF プロキシ）
- ユニット/コンポーネントテスト（Vitest + React Testing Library）
- `web/Dockerfile`・`web/.dockerignore`（Cloud Run デプロイ用イメージ定義）

### 含まない（Phase 2 以降へ送る — §9 参照）

- バックエンド API の変更（CORS 追加、status 公開、再生位置 API、設定 API 等）
- Podcast 生成ステータスのポーリング表示（API が `status` を返さないため不可能）
- 再生位置のサーバー同期（`PATCH /podcasts/:id/position` が存在しないため localStorage のみ）
- デフォルト難易度のサーバー保存（`PUT /settings` が存在しない）
- E2E テスト（Playwright）、Web Push、オフライン再生、Cloud Run への実デプロイ作業

---

## 2. 設計書と実 API の差分と決定事項

**ここが本計画の核心。** `web-design.html` 作成後にバックエンドが確定したため、以下の乖離がある。Coder は迷わず下表の「決定」に従うこと。

| # | 設計書の記述 | 実 API（正） | 決定 |
|---|---|---|---|
| D1 | `GET/POST/DELETE /subscriptions/:id`（id・enabled トグルあり） | `GET/POST /settings/sources`、`DELETE /settings/sources?url=`。ソースは `{name, url}` のみで **id も enabled も存在しない** | エンドポイントは `/settings/sources` を使用。削除は URL をキーに行う。有効/無効トグルは実装しない |
| D2 | `PATCH /podcasts/:id/position` で再生位置をサーバー保存 | 存在しない | 再生位置は localStorage に `podcast_position:{podcastId}` 形式で保存・復元する。サーバー同期は Phase 2 |
| D3 | `GET /settings` / `PUT /settings` でデフォルト難易度・速度を保存 | 存在しない | デフォルト再生速度は localStorage のみに保存。**難易度選択 UI は実装しない**（難易度はバッチ側の環境変数で決まり、API から変更不可のため） |
| D4 | Podcast の `status`（processing/…）表示と 5 秒ポーリング | `PodcastResponse` に `status`・`playback_position_seconds` フィールドは**含まれない**（Firestore モデルには存在するが API 非公開） | ステータスバッジ・ポーリングは実装しない。代わりに一覧画面に手動リフレッシュボタンを置く。`StatusBadge`・`usePodcastPolling` は**作らない** |
| D5 | Star クリック →「Podcast 生成開始」トースト | Star は `starred_article_ids` への追加のみ。生成は**日次バッチ**（podcast-generator Job）で行われる | トースト文言は「Star しました。Podcast は次回バッチ実行時に生成されます」とする |
| D6 | ブラウザから API へ直接 fetch | バックエンドに **CORS ミドルウェアがなく**、ブラウザからのクロスオリジン fetch は失敗する | Next.js Route Handler による BFF プロキシを実装する（§3.1） |
| D7 | `audio_url` は永続 URL | `GET /podcasts` の `audio_url` は**有効期限 1 時間の署名付き GCS URL** | 再生開始の直前に `GET /podcasts/:id` を呼び直して新鮮な URL を取得してから `<audio>` にセットする |
| D8 | `Podcast.audio_url: string \| null` | 実レスポンスでは常に string | 型定義は `string` とする |
| D9 | `RssSource` に `id`・`enabled` | `{name: string, url: string}` のみ | 型定義は 2 フィールドのみ |
| D10 | エラー 422「RSS フィードとして取得できません」 | `POST /settings/sources` は Pydantic `HttpUrl` 検証のみ（URL 形式不正で 422）。フィード内容の検証はしない | 422 のエラー文言は「URL の形式が正しくありません」とする |

---

## 3. アーキテクチャ決定

### 3.1 BFF プロキシ（CORS 対策）— 最重要決定

**問題:** バックエンド (`backend/api/main.py`) に `CORSMiddleware` がない。Web アプリ（Cloud Run 別サービス）のブラウザ JS から API へ直接 fetch するとプリフライトで失敗する。

**決定:** `web/app/api/backend/[...path]/route.ts` にキャッチオール Route Handler を実装し、ブラウザは常に同一オリジンの `/api/backend/*` へリクエストする。プロキシはサーバー側（Next.js standalone サーバー）でバックエンドへ転送するため CORS が発生しない。

**採らなかった代替案とその理由:**
- *バックエンドに CORSMiddleware を追加*: backend の変更・再デプロイが必要になり、「1 ブランチ = 1 目的」の原則に反して web 実装がバックエンド作業をブロック要因に持つ。Phase 2 で CORS を追加すればプロキシは薄いまま除去も可能。
- *Next.js rewrites*: 転送先 URL がビルド時固定になる。本アプリは API ベース URL をユーザーが実行時に設定する設計（localStorage）のため不適。

**プロキシの契約:**
- ブラウザ側 `lib/api.ts` は、リクエストヘッダーに `X-API-Key`（ユーザー入力のキー）と `X-Backend-Base-Url`（ユーザー入力の API ベース URL）を付けて `/api/backend/{path}` を呼ぶ。
- プロキシは `X-Backend-Base-Url` の値と path・クエリ文字列を結合してバックエンドへ転送。`X-API-Key` ヘッダーと `Content-Type`、リクエストボディ、HTTP メソッドをそのまま引き継ぐ。
- バックエンドのステータスコードとレスポンスボディを**加工せず**そのまま返す（エラーハンドリングはクライアント側 `ApiError` の責務）。
- `X-Backend-Base-Url` が欠落・不正（`http(s)://` 以外）の場合は 400 を返す。**SSRF 緩和のため `http://`・`https://` スキーム以外は拒否する**。
- GET/POST/PATCH/DELETE を export する（PATCH は将来用だが対称性のため用意してよい）。

### 3.2 状態管理

- **グローバル（React Context + useReducer、`contexts/AppContext.tsx`）**: API 設定（baseUrl / apiKey / isConfigured）、デフォルト再生速度、音声再生状態（currentPodcast / isPlaying / currentTime / duration / playbackSpeed）。
- **ローカル（各ページの useState/useReducer）**: articles / podcasts / sources の一覧、isLoading、error。
- ライブラリ（Zustand 等）は導入しない。この規模では Context で十分（設計書どおり）。
- localStorage アクセスは `hooks/useLocalStorage.ts` に集約し、SSR 時（`window` 不在）に安全であること。**ページコンポーネントから直接 `localStorage` を触らない。**

### 3.3 レンダリング方針

- データ取得を伴うページはすべて `'use client'`。API キーが localStorage（クライアント）にある以上、Server Components でのデータ取得は不可能。Next.js は「ルーティング・ビルド・配信基盤」として使う。
- これにより hydration 不整合の温床を避ける。初回マウント前に localStorage を読まない（`useEffect` 内で読む）こと。

### 3.4 テスト戦略（TDD 必須）

- **ツール**: Vitest + React Testing Library + jsdom + @testing-library/user-event。`npm run test` で全テストが 10 秒以内に完走すること。
- **方針**: 各タスクで先に失敗するテストを書く（本書の各タスクに「テスト観点」として Red リストを明記）。fetch は `vi.stubGlobal('fetch', vi.fn())` でモック。`Audio` は jsdom に実装がないためモッククラスを `tests/helpers/` に用意する。
- **テスト配置**: `web/tests/` 配下にソースと同じ階層構造（例: `tests/lib/api.test.ts`, `tests/components/ArticleCard.test.tsx`）。
- 必須パターン（agent-rules/11）: エラーハンドリング・境界値・インターフェース契約（レスポンス必須フィールドの検証）を各テストに含める。

### 3.5 デザイン方向（agent-rules/15 準拠）

- **美的トーン**: 「ダーク・エディトリアル」。ニュース読み上げアプリとして、夜間・通勤時の利用を想定した低輝度 UI。`web-design.html` のトークンをそのまま CSS 変数として採用する:
  - `--bg: #0f1117` / `--surface: #1a1d27` / `--border: #2a2d3e` / `--text: #e2e4ed` / `--muted: #8b8fa8` / `--accent: #7c6af7` / `--accent2: #4fc3f7` / `--green: #4caf82` / `--orange: #f0a050` / `--red: #e05c5c`
- **フォント**（禁止リスト回避: Inter / Roboto / Arial / system-ui / Space Grotesk は使用不可）:
  - 見出し（欧文）: **Bricolage Grotesque**（next/font/google）
  - 本文（和文）: **Zen Kaku Gothic New**
  - メタ情報・数値（時間表示等）: **IBM Plex Mono**
- **モーション**: ページロード時のカードのスタッガードリビール（CSS のみ、`animation-delay` をインデックスで階段化）と、Star/Dismiss 時のカード除去トランジションに集中投資。常時動くアニメーションは置かない。
- **レイアウト**: 単調なセンター 1 カラムを避け、Feed はスコアバーを左端に縦帯で配置するなど非対称要素を入れる。背景は単色でなく微細なノイズ/グラデーションメッシュを 1 枚かける。

### 3.6 エラーハンドリング共通方針

- `lib/api.ts` に `ApiError extends Error`（`status: number`, `detail: string`）を定義。FastAPI のエラーボディは `{"detail": "..."}` 形式。JSON でないボディは `detail = 'Unknown error'` にフォールバック。
- 401 はグローバルに「API キーが正しくありません。設定を確認してください」のトースト + `/settings` への誘導。
- ネットワーク断（fetch reject）は `ApiError(0, ...)` に正規化し「サーバーに接続できません」を表示。
- API キーや署名付き URL を `console.log` しない（agent-rules/12）。

---

## 4. API 契約（実装済みバックエンドからの転記 — これが正）

ベース URL はユーザー設定（Cloud Run の API サービス URL）。全エンドポイント（`/health` 以外）は `X-API-Key` ヘッダー必須。不一致は `401 {"detail": "Invalid or missing API key"}`。

### GET /feed
```json
{
  "articles": [
    {"id": "abc123", "title": "...", "url": "https://...", "source": "HackerNews",
     "score": 0.95, "published_at": "2026-06-10T00:00:00+00:00"}
  ],
  "date": "2026-06-10"
}
```
- 当日のレコメンドが未生成なら `articles: []`（200）。最大 50 件。score 降順で返る。

### POST /articles/{id}/star ・ POST /articles/{id}/dismiss
```json
{"status": "starred", "article_id": "abc123"}
```
- 記事が存在しない場合 404。冪等（再 Star しても 200）。
- **dismiss してもサーバーは当日の feed から除去しない**（翌日以降の推薦から除外）。当日のリストからの除去はクライアントの責務。

### GET /podcasts ・ GET /podcasts/{id}
```json
{
  "podcasts": [
    {"id": "uuid", "type": "single", "article_ids": ["abc123"], "difficulty": "toeic_900",
     "audio_url": "https://storage.googleapis.com/...(署名付き・1時間有効)",
     "japanese_intro_text": "...", "duration_seconds": 300,
     "created_at": "2026-06-10T00:00:00+00:00"}
  ]
}
```
- `status` フィールドは**存在しない**（D4）。`created_at` 降順、最大 50 件。単体取得は存在しなければ 404。

### GET /settings/sources ・ POST /settings/sources ・ DELETE /settings/sources?url={url}
```json
{"sources": [{"name": "HackerNews", "url": "https://news.ycombinator.com/rss"}]}
```
- POST ボディ: `{"name": "...", "url": "https://..."}`。URL 重複は 409、URL 形式不正は 422（Pydantic HttpUrl）。
- DELETE はクエリパラメータ `url` で対象指定（URL エンコード必須）。存在しなければ 404。
- POST/DELETE とも**更新後の全 sources を返す**ので、レスポンスでそのまま一覧 state を置き換えればよい（再 GET 不要）。

### 型定義の正（`web/types/index.ts` に定義すべき形）

```
Difficulty = 'toeic_600' | 'toeic_900' | 'ielts_55' | 'ielts_7' | 'eiken_2' | 'eiken_p1'
Article    = { id, title, url, source: string; score: number; published_at: string }
Podcast    = { id: string; type: 'single' | 'digest'; article_ids: string[];
               difficulty: Difficulty; audio_url: string; japanese_intro_text: string;
               duration_seconds: number; created_at: string }
RssSource  = { name: string; url: string }
FeedResponse / PodcastListResponse / RssSourcesResponse / ActionResponse は上記 JSON と同形
```

---

## 5. ファイル構成（成果物の全体像）

```
web/
├── app/
│   ├── layout.tsx                 # ルートレイアウト（Provider + NavigationBar + AudioPlayerBar）
│   ├── globals.css                # デザイントークン（CSS 変数）・フォント・共通アニメーション
│   ├── page.tsx                   # / : 設定済みなら /feed へ redirect、未設定なら SetupModal
│   ├── feed/page.tsx
│   ├── podcast/page.tsx
│   ├── podcast/[id]/page.tsx
│   ├── subscriptions/page.tsx
│   ├── settings/page.tsx
│   └── api/backend/[...path]/route.ts   # BFF プロキシ（§3.1）
├── components/
│   ├── NavigationBar.tsx
│   ├── AudioPlayerBar.tsx
│   ├── ArticleCard.tsx
│   ├── PodcastCard.tsx
│   ├── subscriptions/SubscriptionRow.tsx
│   ├── subscriptions/AddSubscriptionForm.tsx
│   ├── providers/AppProvider.tsx
│   └── ui/  DifficultyBadge.tsx / Toast.tsx / ConfirmDialog.tsx / SetupModal.tsx / SkeletonCard.tsx
├── contexts/AppContext.tsx
├── hooks/  useAudioPlayer.ts / useLocalStorage.ts
├── lib/    api.ts / config.ts / format.ts
├── types/index.ts
├── tests/  （src と同階層構造 + helpers/mockAudio.ts）
├── Dockerfile / .dockerignore
├── next.config.ts（output: 'standalone'）/ vitest.config.ts / tsconfig.json / package.json
```

設計書からの構成変更: `StatusBadge.tsx`・`hooks/usePodcastPolling.ts` は作らない（D4）。`lib/format.ts`（秒 → MM:SS、ISO 日時 → 相対表示）を追加。

---

## 6. タスク分解（この順序で実装すること）

依存関係: T1 → T2 → T3 → T4 → T5 → T6 →（T7〜T8）→ T9 → T10 →（T11 → T12 → T13 → T14）→ T15 → T16 → T17。括弧内も記載順を推奨。

---

### Task 1: プロジェクト雛形とテスト基盤

**目的:** 以降の全タスクが TDD で進められる土台を作る。
**ファイル:** `web/package.json`, `next.config.ts`, `tsconfig.json`, `vitest.config.ts`, `app/layout.tsx`(仮), `app/page.tsx`(仮), `app/globals.css`(仮)

- [ ] `create-next-app`（TypeScript / Tailwind / App Router / src ディレクトリなし）で `web/` に生成
- [ ] `next.config.ts` に `output: 'standalone'` を設定（Dockerfile が前提とするため必須）
- [ ] Vitest + React Testing Library + jsdom + user-event を devDependencies に追加し、`vitest.config.ts`（environment: jsdom, globals: true, tsconfig paths 解決）と `npm run test` スクリプトを設定
- [ ] 動作確認用の自明なテスト（例: layout が children を描画する）を 1 件作成し `npm run test` がパスすることを確認
- [ ] `npm run build` が成功することを確認
- [ ] コミット分割: ①雛形生成（`構成: Next.js プロジェクト雛形を生成`）②standalone 設定 ③テスト基盤（`構成: Vitest によるテスト基盤を導入`）

**完了条件:** `npm run test` / `npm run build` / `npm run dev` が全て成功。

---

### Task 2: 型定義と localStorage 設定基盤

**目的:** API 契約（§4）を型として固定し、設定永続化の唯一の入口を作る。
**ファイル:** `types/index.ts`, `lib/config.ts`, `hooks/useLocalStorage.ts` + 対応テスト

- [ ] **Red:** `useLocalStorage` のテストを先行作成。観点: 初期値の返却 / set 後の永続化と再読込 / `window` 不在（SSR）で例外を出さない / JSON 不正値が保存されていた場合に初期値へフォールバック
- [ ] `types/index.ts` に §4 の型を定義（フィールド名は **snake_case のまま**。API レスポンスを変換せず使うため。WHY: 変換層を挟むとバックエンド変更時の追従コストが上がる）
- [ ] `lib/config.ts` に localStorage キー定数を集約: `api_base_url`, `api_key`, `default_playback_speed`, `podcast_position:{id}`（プレフィックス定数 + キー生成関数）
- [ ] **Green → Refactor → コミット**（テストと実装は別コミット: `テスト: useLocalStorage の仕様を先行定義` → `機能: 型付き localStorage フックと設定キー定数を実装`）

**完了条件:** 型が §4 と 1:1 対応。localStorage アクセスがこのフック経由に一本化される設計が確立。

---

### Task 3: API クライアント `lib/api.ts`

**目的:** 全画面が使う通信層。エラー正規化をここで完結させる。
**ファイル:** `lib/api.ts` + `tests/lib/api.test.ts`

- [ ] **Red:** テスト観点を先行作成:
  - 各関数が正しいパス・メソッド・ヘッダー（`X-API-Key`, `X-Backend-Base-Url`, `Content-Type`）で `/api/backend/*` を fetch する
  - `deleteSource` が URL を `encodeURIComponent` してクエリに載せる（境界値: `&` や日本語を含む URL）
  - 非 2xx → `ApiError(status, detail)` を throw（`{"detail": "..."}` 抽出 / JSON でないボディは 'Unknown error'）
  - fetch reject → `ApiError(0, ...)` に正規化
  - 契約検証: `getFeed` の戻り値に `articles`・`date` が存在する
- [ ] 実装する関数（シグネチャ契約）: `getFeed()` / `starArticle(id)` / `dismissArticle(id)` / `getPodcasts()` / `getPodcast(id)` / `getSources()` / `addSource(name, url)` / `deleteSource(url)`
  - 設計書にあった `updatePosition` / `getSettings` / `updateSettings` / `getSubscriptions` 系は**作らない**（D2/D3/D1）
  - ベース URL と API キーは引数または生成関数（例: `createApiClient(config)`) で注入し、モジュールロード時に localStorage を読まない（WHY: SSR 安全性とテスト容易性。設計書の「モジュールトップで localStorage 読込」は採用しない）
- [ ] **Green → Refactor → コミット**

**完了条件:** 全 API 関数がテストで契約検証済み。localStorage 非依存（呼び出し側が注入）。

---

### Task 4: BFF プロキシ Route Handler

**目的:** CORS 問題の解決（§3.1）。
**ファイル:** `app/api/backend/[...path]/route.ts` + `tests/app/api/proxy.test.ts`

- [ ] **Red:** テスト観点: GET/POST/DELETE の転送（path・クエリ・ボディ・`X-API-Key` 引継ぎ）/ バックエンドの 401・404・409 ステータスとボディの素通し / `X-Backend-Base-Url` 欠落 → 400 / `ftp://` 等の不正スキーム → 400 / クエリ文字列がエンコードを保って転送される
- [ ] Route Handler を実装（Request/Response は Web 標準 API なので jsdom 環境のままテスト可能）
- [ ] **Green → Refactor → コミット**（`機能: CORS 回避のためのバックエンドプロキシを実装`）

**完了条件:** ローカルで `npm run dev` を起動し、`curl localhost:3000/api/backend/health -H "X-Backend-Base-Url: <実APIのURL>"` が `{"status":"ok"}` を返す（手動確認、結果をコミットメッセージ等に残す必要はない）。

---

### Task 5: AppContext / AppProvider

**目的:** API 設定と再生状態のグローバル管理。
**ファイル:** `contexts/AppContext.tsx`, `components/providers/AppProvider.tsx` + テスト

- [ ] **Red:** テスト観点: 初期状態は未設定（`isConfigured: false`）/ `configure(baseUrl, key)` で localStorage に永続化され `isConfigured: true` / マウント時に localStorage から復元 / 再生状態 action（SET_PODCAST / PLAY / PAUSE / SET_TIME / SET_SPEED）の reducer 遷移 / 復元前に `isConfigured` が `false` のまま（hydration 安全）
- [ ] AppState は §3.2 の構造。reducer + Context + カスタムフック `useApp()`（Provider 外で throw）で公開
- [ ] **Green → Refactor → コミット**

**完了条件:** 以降の画面タスクが `useApp()` だけでグローバル状態へアクセスできる。

---

### Task 6: デザイントークン・ルートレイアウト・NavigationBar

**目的:** 全画面の骨格と視覚言語の確立（§3.5）。
**ファイル:** `app/globals.css`, `app/layout.tsx`, `components/NavigationBar.tsx` + テスト

- [ ] **Red:** NavigationBar テスト: 4 リンク（Feed/Podcast/Subscriptions/Settings）が描画される / 現在パスのリンクに `aria-current="page"` が付く
- [ ] `globals.css` に §3.5 の CSS 変数・フォント・スタッガードリビール用 keyframes を定義。next/font で Bricolage Grotesque / Zen Kaku Gothic New / IBM Plex Mono を読み込み CSS 変数に紐付け
- [ ] `layout.tsx`: `<AppProvider>` → NavigationBar（上部固定）→ `<main>` → AudioPlayerBar 用スロット（Task 12 まではプレースホルダーなし、レイアウト下部余白のみ確保）
- [ ] モバイル（375px）で NavigationBar が下部タブ化するレスポンシブ対応（PRD のユースケースは通勤中のスマホ利用）
- [ ] **Green → コミット**（CSS とコンポーネントは別コミット）

**完了条件:** 全ページ共通の枠が完成し、`npm run dev` でダークテーマと和欧フォントが適用されている。

---

### Task 7: 汎用 UI コンポーネント

**目的:** 画面タスクが組み立てに専念できる部品を先に揃える。
**ファイル:** `components/ui/Toast.tsx`, `DifficultyBadge.tsx`, `SkeletonCard.tsx`, `ConfirmDialog.tsx`, `lib/format.ts` + 各テスト

- [ ] **Red → Green を部品ごとに小さく回す**。テスト観点:
  - Toast: メッセージ表示 / 3 秒後に自動消滅（`vi.useFakeTimers`）/ success・error の見た目区別（role="status" / role="alert"）。Context ベースの `useToast()` で全画面から呼べること
  - DifficultyBadge: 6 難易度すべてに表示ラベル（例: `toeic_900` → 「TOEIC 900」）と色クラスを割当。**未知の文字列でも例外を出さず生値を表示**（境界値）
  - ConfirmDialog: 開閉 / 確認・キャンセルのコールバック / Escape で閉じる
  - format.ts: `formatDuration(300)` → `"5:00"` / 0 秒 / 3600 秒以上 / `formatDate(ISO文字列)` → 「6/10 09:00」形式。不正 ISO 文字列で例外を出さない
- [ ] コミットは部品単位でアトミックに（例: `機能: 操作フィードバック用トースト通知を実装`）

**完了条件:** 全部品が単体テスト済みで、Storybook 的な依存なしに画面から import 可能。

---

### Task 8: SetupModal と / （エントリーページ）

**目的:** 初回利用者が API 接続情報を設定するまで他画面を使わせないゲート。
**ファイル:** `components/ui/SetupModal.tsx`, `app/page.tsx` + テスト

- [ ] **Red:** テスト観点: 未設定時にモーダルが表示される / baseUrl・apiKey を入力して保存 → `configure()` が呼ばれる / baseUrl が `https://` で始まらない入力はインラインエラー（保存不可）/ 空入力で保存不可 / API キー入力は `type="password"`
- [ ] `app/page.tsx`: `isConfigured` なら `/feed` へ `router.replace`、未設定なら SetupModal を表示。判定は復元完了後に行う（チラつき防止のため復元前はスケルトン表示）
- [ ] SetupModal には「接続テスト」ボタンを置き、`/api/backend/health` を叩いて成否をインライン表示する（WHY: キー誤入力に最初に気付ける場所がここしかない。`/health` は認証不要なので **baseUrl の疎通確認**として機能する。キー検証は `GET /feed` を呼び 401 か否かで行う）
- [ ] **Green → Refactor → コミット**

**完了条件:** 未設定 → 設定 → /feed 遷移のフローがテストで保証される。

---

### Task 9: ArticleCard

**目的:** Feed の中核部品。Star/Dismiss の操作性がアプリの第一印象を決める。
**ファイル:** `components/ArticleCard.tsx` + テスト

- [ ] **Red:** テスト観点: タイトル・ソース・公開日・スコアバー（width がスコア比例、`aria-valuenow` 付き）の描画 / タイトルは `target="_blank"` + `rel="noopener noreferrer"` の外部リンク / ★ クリック → `onStar(id)` / × クリック → `onDismiss(id)` / 操作中はボタンが disabled（二重送信防止）/ score 0 と 1.0 の境界値描画
- [ ] props は `{ article, onStar, onDismiss, busy }` の制御コンポーネントとする（API 呼び出しはページの責務。WHY: カード自体を純粋に保ちテストを軽くする）
- [ ] **Green → Refactor → コミット**

---

### Task 10: /feed ページ

**目的:** 記事一覧の取得・Star/Dismiss・各種状態表示。
**ファイル:** `app/feed/page.tsx` + テスト

- [ ] **Red:** テスト観点:
  - マウント時に `getFeed` を呼び記事カードを描画 / ローディング中は SkeletonCard
  - 空配列 → 「まだ記事がありません。バッチは毎日 06:00 に実行されます。」
  - Star 成功 → トースト「Star しました。Podcast は次回バッチ実行時に生成されます」（D5）。**カードはリストに残す**（Star は非表示化ではない）が ★ を点灯状態にする
  - Dismiss 成功 → カードをリストから即時除去（D5 注記: クライアント責務）
  - Star/Dismiss の 404 → 「記事が見つかりません」トースト + カード除去 / 401 → 設定誘導トースト
  - リフレッシュボタンで再取得
- [ ] 楽観更新はしない（先に API 成功を待ってから state 更新。WHY: 個人利用でレイテンシ要件が緩く、ロールバック実装の複雑さに見合わない）
- [ ] **Green → Refactor → コミット**（ページ実装とテストは別コミット）

**完了条件:** 実 API（または dev サーバー + 実バックエンド）で一覧表示と Star/Dismiss が動くことを手動確認。

---

### Task 11: useAudioPlayer フック

**目的:** HTML5 Audio のラッパー。再生ロジックを UI から完全分離する。
**ファイル:** `hooks/useAudioPlayer.ts`, `tests/helpers/mockAudio.ts` + テスト

- [ ] まず `tests/helpers/mockAudio.ts` に `Audio` モック（play/pause/currentTime/playbackRate/イベント発火ヘルパー）を作る
- [ ] **Red:** テスト観点:
  - `load(url, resumePosition)` → `currentTime` が復元される
  - play/pause/seek(±秒)/setSpeed の各操作がオーディオ要素へ反映される
  - `timeupdate` ごとに現在位置が更新され、**10 秒間隔のスロットルで** `podcast_position:{id}` へ localStorage 保存される（fake timers）
  - `ended` → isPlaying=false、保存位置を 0 にリセット
  - 別エピソードの `load` やアンマウント → 旧 Audio の pause と `src` 解放（リーク防止）
- [ ] 速度は 8 段階 `[0.5, 0.75, 1.0, 1.25, 1.5, 1.75, 2.0, 2.5]` を定数として export
- [ ] **Green → Refactor → コミット**

**完了条件:** UI なしでフック単体の振る舞いが全て検証済み。

---

### Task 12: AudioPlayerBar（全画面共通の再生バー）

**目的:** 画面下部固定の再生 UI。AppContext の再生状態と useAudioPlayer を結合する。
**ファイル:** `components/AudioPlayerBar.tsx`, `app/layout.tsx`（組込み）+ テスト

- [ ] **Red:** テスト観点: `currentPodcast` が null なら非表示 / イントロ先頭 50 文字 + DifficultyBadge 表示 / 再生・一時停止トグル / -15 秒・+30 秒ボタン / シークバー（`input[type=range]`、`aria-label` 付き）操作で seek / 速度セレクタが 8 段階で、初期値は AppContext のデフォルト速度 / 時間表示が `lib/format.formatDuration` 形式
- [ ] layout.tsx に組み込み、`<main>` の下端 padding をバー高さぶん確保（コンテンツ隠れ防止）
- [ ] **Green → Refactor → コミット**

---

### Task 13: PodcastCard と /podcast 一覧ページ

**目的:** エピソード一覧と再生開始。署名付き URL の失効対策（D7）をここで実装する。
**ファイル:** `components/PodcastCard.tsx`, `app/podcast/page.tsx` + テスト

- [ ] **Red:** PodcastCard テスト観点: イントロ先頭 80 文字・DifficultyBadge・`formatDuration(duration_seconds)`・生成日の描画 / 再生ボタン → `onPlay(podcast)` / カード本体クリック → `/podcast/:id` へのリンク / 保存済み再生位置があれば「続きから MM:SS」表示
- [ ] **Red:** ページテスト観点:
  - `getPodcasts` で一覧描画 / 空 → 「Podcast がまだありません。Feed で記事を Star すると次回バッチで生成されます。」/ ローディング → Skeleton / リフレッシュボタン
  - **再生ボタン押下時: `getPodcast(id)` を呼び直して新鮮な `audio_url` を取得してから AppContext へ SET_PODCAST + 再生開始**（D7。一覧取得時の URL は使わない）
  - 再生開始時に `podcast_position:{id}` から復元位置を渡す
- [ ] ステータス表示・ポーリングは実装しない（D4）
- [ ] **Green → Refactor → コミット**（カードとページは別コミット）

---

### Task 14: /podcast/[id] 詳細ページ

**目的:** イントロ全文の閲覧と再生。
**ファイル:** `app/podcast/[id]/page.tsx` + テスト

- [ ] **Red:** テスト観点: `getPodcast(id)` で取得し日本語イントロ**全文**・難易度・時間・生成日・元記事リンク（article_ids は ID のみなので「元記事 ID」をテキスト表示で可）を描画 / 404 → 「エピソードが見つかりません」+ 一覧へ戻るリンク / 再生ボタンは Task 13 と同じフロー（このページでも取得直後の URL で再生開始してよい）
- [ ] **Green → Refactor → コミット**

---

### Task 15: /subscriptions（RSS 購読管理）

**目的:** ソース一覧・追加・削除。設計書の `/subscriptions` API ではなく `/settings/sources` を使う（D1）。
**ファイル:** `components/subscriptions/AddSubscriptionForm.tsx`, `SubscriptionRow.tsx`, `app/subscriptions/page.tsx` + テスト

- [ ] **Red:** フォームテスト観点: name・url の 2 入力 / クライアント側 URL 形式チェック（`http(s)://` 始まり。不正ならインラインエラーで送信しない）/ 送信成功 → 入力クリア / 失敗（409）→ 「この URL は登録済みです」をインライン表示し**入力値は保持** / 422 → 「URL の形式が正しくありません」（D10）/ 送信中は disabled
- [ ] **Red:** ページテスト観点: `getSources` で一覧描画 / 追加・削除のレスポンス（更新後全件）で state を**置き換える**（再 GET しない。§4 参照）/ 削除ボタン → ConfirmDialog → 確認後 `deleteSource(url)` / 削除 404 → 「対象が見つかりません」トースト + 再取得 / 空 → 「購読ソースがありません」と追加フォームへの誘導
- [ ] enabled トグル・行 ID は実装しない（D1/D9）
- [ ] **Green → Refactor → コミット**（フォーム・行・ページで 3 コミット目安）

---

### Task 16: /settings ページ

**目的:** API 接続設定の変更とデフォルト再生速度（D3 により**この 2 種のみ**）。
**ファイル:** `app/settings/page.tsx` + テスト

- [ ] **Red:** テスト観点: 現在の baseUrl 表示・apiKey はマスク表示（値は出さない、「設定済み」表示 + 再入力欄）/ 保存 → AppContext の `configure()` 経由で localStorage 更新 / デフォルト再生速度セレクタ（8 段階、Task 11 の定数を import）→ localStorage 保存 + AppContext 反映 / 接続テストボタン（Task 8 と同一ロジックを共通化して再利用）
- [ ] 難易度設定 UI は置かない。代わりに「Podcast の難易度はサーバー側設定で管理されています」の説明文を表示（WHY: ユーザーが設定箇所を探して迷うのを防ぐ）
- [ ] **Green → Refactor → コミット**

---

### Task 17: Dockerfile・デプロイ準備

**目的:** Cloud Run デプロイ可能なイメージ定義。**デプロイ実行は本計画のスコープ外**（backend Task 13 と同方針）。
**ファイル:** `web/Dockerfile`, `web/.dockerignore`

- [ ] 設計書 §9 のマルチステージ Dockerfile（node:22-alpine、standalone 出力をコピー、EXPOSE 3000）を作成。**非 root ユーザーで実行する**（backend Dockerfile.api と同方針: `USER node` 等。WHY: PR #1 で API コンテナに適用済みのセキュリティ基準に合わせる）
- [ ] `.dockerignore`: `node_modules`, `.next`, `tests`, `*.test.*`, `.env*`
- [ ] `docker build` がローカルで成功し、`docker run -p 3000:3000` で起動して `/` が表示されることを確認
- [ ] コミット: `構成: Cloud Run デプロイ用の Web コンテナ定義を追加`
- [ ] デプロイコマンド（設計書 §9 のもの）は README または `web/README.md` に転記しておく（実行はしない）

---

## 7. 受け入れ基準（全タスク完了後の検証）

1. `cd web && npm run test` — 全テストパス、10 秒以内
2. `npm run build` — 型エラー・lint エラーなし
3. 手動シナリオ（実バックエンドに対して dev サーバーで確認）:
   - 初回アクセス → SetupModal → 接続テスト成功 → /feed 表示
   - 記事を Star → トースト表示・★ 点灯 / Dismiss → カード消失
   - /podcast で再生 → AudioPlayerBar 表示 → 速度変更・±シーク → ページ遷移しても再生継続
   - リロード後に同エピソードを再生 → 前回位置から再開
   - /subscriptions で追加（重複 409 の文言確認）・削除（確認ダイアログ）
   - 誤った API キーに変更 → 401 トーストが表示される
4. `docker build` 成功・コンテナ起動確認
5. API キー・署名付き URL がコンソールログ・コミットに含まれていない

---

## 8. Git 運用

- 作業ブランチ: `dev/web`（バックエンドの `dev/backend` → PR #1 と同じ運用）。タスク単位でさらに分ける場合は `feature/web-<task>` を `dev/web` から切る
- コミット: agent-rules/10 準拠。日本語 1〜2 文・`<種別>: <説明>`・WHY 優先・`git add .` 禁止・テストと実装は別コミット・Co-Authored-By 等のメタデータ禁止
- 各タスク完了時点でテストが全てグリーンであること（Red 状態のままタスクをまたがない）
- 全タスク完了後、`main` への PR を作成（マージ判断は人間レビュー）

---

## 9. Phase 2 送り（本計画では着手禁止）

| 項目 | 必要なバックエンド変更 |
|---|---|
| Podcast 生成ステータス表示・ポーリング | `PodcastResponse` への `status`/`error_message` 追加 |
| 再生位置のサーバー同期（端末間共有） | `PATCH /podcasts/:id/position` 新設 |
| デフォルト難易度のユーザー設定 | `GET/PUT /settings` 新設 + podcast-generator の difficulty を UserPrefs 参照に変更 |
| BFF プロキシの撤去（直接 fetch 化） | backend への CORSMiddleware 追加 |
| Web Push 通知 / オフライン再生（Service Worker） | 通知基盤の新設 |
| E2E テスト（Playwright） | なし（web 単独で追加可能） |

これらに触れたくなった場合は実装せず、計画の更新を提案すること。
