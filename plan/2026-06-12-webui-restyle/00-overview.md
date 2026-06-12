# WebUI リスタイル実装計画（オーケストレーター向け統括指示書）

> **For orchestrator (Main Claude):** 本計画は auto_mode の coder サブエージェントを**並列起動**して実行する前提で設計されている。各タスク指示書（`01`〜`11`）はそれぞれ 1 体の coder に独立して渡せる自己完結文書である。本書はオーケストレーターが Wave 管理・品質ゲート・コミット統合を行うための統括文書。

**作成日:** 2026-06-12
**作業ブランチ:** `feature/webui-restyle`（`main` から作成。coder は git 操作禁止、コミットは git-composer の責務）
**Goal:** Docker で稼働中のローカル Web アプリ（`web/` 配下、Next.js 15）に、デザイン済み WebUI `docs/design/app-ui.html` を反映する。機能（API 連携・再生・購読管理）は既存実装を維持し、視覚言語・レイアウト・テーマ切替を新デザインに置き換える。

---

## 0. 前提・参照ドキュメント

| ドキュメント | 用途 |
|---|---|
| `docs/design/app-ui.html` | **反映すべきデザインの正本**（2,144 行の静的モック。CSS・マークアップ・挙動デモ） |
| `docs/plan/2026-06-10-web-frontend.md` | 既存実装の計画書。**§2 の D1〜D10（API との乖離決定）は引き続き有効** |
| `docs/spec/2026-06-10-web-frontend-spec.md` | 機能仕様の正本（本計画では機能変更しないため挙動の判断基準） |
| `agent-rules/15-frontend-design.md` | デザインガイドライン（禁止フォント: Inter / Roboto / Arial / system-ui / Space Grotesk） |
| `agent-rules/11-testing-strategy.md` | TDD 必須 |
| `agent-rules/91-claude-subagent-coding.md` | サブエージェント運用プロトコル |

**現状の重要な事実（調査済み・2026-06-12 時点）:**
- `web/` には **globals.css が存在せず、Tailwind も未導入**。コンポーネントは素のセマンティック HTML + 最小限のクラス名のみ
- 既存テスト（Vitest + RTL）は **role / aria-* / テキストベース**で書かれており、クラス名追加・スタイル変更では壊れない設計
- テーマ切替機構は存在しない。`next/font` も未使用
- Docker 起動: `docker compose up --build api web`（リポジトリルート）。Web は `web/Dockerfile.web`、`next.config.ts` は standalone 出力

---

## 1. スコープ

### 含む
- `app/globals.css` 新設（app-ui.html の CSS 全量移植 + デザイントークン）
- `next/font/google` によるフォント導入（Playfair Display / DM Sans / DM Mono — 禁止リスト非該当）
- ダーク/ライトテーマ切替（`html[data-theme]` + localStorage + `prefers-color-scheme`）
- サイドバーレイアウト（app-shell グリッド）への再構成
- 全コンポーネント・全ページのリスタイル（クラス名付与 + デザイン準拠マークアップ）
- 新規挙動: テーマトグル / フィードタブ（クライアントフィルタ）/ 再生中カード強調 / おすすめソース

### 含まない
- バックエンド API の変更、API クライアント（`lib/api.ts`）・BFF プロキシの変更
- `contexts/`・`hooks/useAudioPlayer.ts`・`hooks/useLocalStorage.ts`・`hooks/useStartPodcast.ts` のロジック変更
- Tailwind / CSS-in-JS の導入（純 CSS で完結。WHY: デザイン正本が純 CSS であり移植が最短・依存追加ゼロ・既存テスト無風）
- E2E テスト（Playwright）導入

---

## 2. デザインと実装の差分決定（D11〜D27）

**ここが本計画の核心。** app-ui.html はモックであり、バックエンドに存在しない機能の UI を含む。coder は迷わず下表に従うこと（旧計画 D1〜D10 と整合）。

| # | デザインの記述 | 決定 | 理由 |
|---|---|---|---|
| D11 | Podcast カードの「生成中/完了/失敗」バッジ・「あと約1分」 | **実装しない**（完了バッジも出さない） | API レスポンスに `status` がない（旧 D4） |
| D12 | Podcast 画面の「新規生成」ボタン | **実装しない** | 生成は日次バッチのみ（旧 D5） |
| D13 | 購読ソースの有効/無効トグル | **実装しない** | API に `enabled` がない（旧 D1） |
| D14 | 設定画面の「デフォルト難易度」セレクト | **実装しない**。既存の説明文（サーバー側設定で管理）を維持 | API がない（旧 D3） |
| D15 | フィードタブ「未読」 | **実装しない**。タブは「すべて」「★ スター済み」の 2 つ（クライアント側フィルタ + 件数表示） | 既読管理 API がない。スター済みは既存の `starredIds` state で実現可能 |
| D16 | フィードヘッダー「全部スターする」 | **実装しない**（Phase 2） | N 回 API 呼び出しの失敗ハンドリングが本計画のスコープを超える |
| D17 | サイドバーのフィードバッジ（未読数 12） | **実装しない** | 未読概念がなく、ページ外で件数を持つグローバル状態追加に見合わない |
| D18 | サイドバー「バッチ: 正常稼働中」ステータスドット | **実装しない**。フッターはテーマトグルのみ | 監視 API がない。嘘の表示は置かない |
| D19 | プレイヤーのシャッフル・音量アイコンボタン | シャッフルは**実装しない**。音量は既存の `input[type=range]` スライダーを維持しスタイルのみ適用 | シャッフル対象（プレイリスト）が存在しない。音量スライダーは仕様 §9 の既存要件 |
| D20 | 速度ピル（クリックでサイクル） | 既存の `<select>` を維持し、`.speed-pill` 風に装飾 | 既存テストとの互換・キーボード操作性。サイクル UI は速度を戻すのに最大 7 クリック必要で UX 劣後 |
| D21 | 設定画面の連続値スピードスライダー（0.5〜2.5） | 既存の 8 段階 `<select>` を維持し装飾 | `PLAYBACK_SPEEDS` 8 段階が既存契約（useAudioPlayer 定数） |
| D22 | 設定画面の「キャッシュサイズ/キャッシュをクリア」 | **実装しない**（Phase 2） | 要件にない。localStorage 全消去は設定喪失の事故リスク |
| D23 | 購読画面「おすすめのソース」（The Verge / dev.to のフォーム自動入力） | **実装する** | クライアント完結で API 不要。空状態からの導線として価値が高い |
| D24 | Podcast カードの「再生中」強調（amber 枠 + 波形） | **実装する** | `AppContext.currentPodcast?.id` との比較で実現可能 |
| D25 | テーマ切替（ダーク/ライト） | **実装する**（本計画唯一の基盤的新機能） | デザインの中核。FOUC 防止のためインラインスクリプトで初期化 |
| D26 | プレイヤーバー常時表示（グリッド第 2 行固定） | `currentPodcast` が null の間は**非表示**（既存挙動を維持）。app-shell のグリッド行は `1fr auto` とし、バー不在時は高さ 0 になる | 再生対象がないのに空バーを出さない（既存テストとの互換） |
| D27 | Star 時トースト「ポッドキャスト生成を開始しました」 | 既存文言「Star しました。Podcast は次回バッチ実行時に生成されます」を**維持** | 生成は即時開始されない（旧 D5）。嘘の文言は出さない |
| D28 | 記事の相対日時表記（「3時間前」） | 既存の `lib/format.formatDate` 出力を維持（Phase 2 で相対化を検討） | `format.ts` は複数タスクから参照され、変更すると並列タスク間の競合源になる |

---

## 3. アーキテクチャ決定

### 3.1 スタイリング: 純 CSS + globals.css 一元管理（最重要・並列化の要）

- **`app/globals.css` は Task 01 が app-ui.html の `<style>` 全量（L17〜L1363 + L2001 の spin）を移植して完成させる。Task 02〜10 は globals.css を編集禁止。**
- コンポーネント側の作業は「デザインのクラス名を付与したマークアップへの変更」のみ。CSS セレクタはすでに存在する前提で書ける
- 不足スタイルを発見した coder は **自分で globals.css を触らず**、成果報告の「残課題・申し送り」に記載 → Task 11（統合）でまとめて調整
- WHY: 9 タスクが同一 CSS ファイルを並列編集するとコンフリクトが必発。デザイン CSS は完成品なので先に全量を固定できる

### 3.2 フォント

- `next/font/google` で読み込み、CSS 変数に紐付け: `--font-display`（Playfair Display）/ `--font-body`（DM Sans）/ `--font-mono`（DM Mono）
- globals.css 移植時、`font-family: 'Playfair Display', serif` 等のリテラルを `var(--font-display)` 等に置換する
- WHY: `<link>` 直書きでなく next/font を使うのは、ビルド時セルフホスティングにより実行時の Google Fonts 依存（Docker コンテナからの外部リクエスト）を消すため

### 3.3 テーマ機構

- `<html data-theme="dark|light">` を正とする（デザインと同一方式）
- FOUC 防止: `app/layout.tsx` の `<head>` 内インラインスクリプトで `localStorage('theme') ?? prefers-color-scheme` を即時適用。`<html suppressHydrationWarning>` を付ける（サーバー出力とクライアント適用値が異なり得るため）
- 切替 UI はサイドバーフッターの `ThemeToggle`（Task 02）。localStorage キーは `lib/config.ts` に `KEY_THEME = 'theme'` として追加（Task 01）
- React state にテーマを持たない（DOM の `data-theme` が単一の真実。WHY: 全コンポーネント再レンダリング不要、CSS 変数だけで切替が完結する）

### 3.4 レイアウト構造

`app/layout.tsx`（Task 01）:

```
<html lang="ja" suppressHydrationWarning>  ← head にテーマ初期化スクリプト
  <body className={フォント変数}>
    <AppProvider><ToastProvider><AudioPlayerProvider>
      <div className="app-shell">          ← grid: サイドバー | メイン / プレイヤー
        <NavigationBar />                   ← aside.sidebar を自身で描画（Task 02）
        <main className="main-content">{children}</main>
        <AudioPlayerBar />                  ← footer.player-bar を自身で描画（Task 06）
      </div>
    （Provider 閉じ）
```

- `body { overflow: hidden }`、スクロールは `.main-content` が担う（デザイン準拠）
- 各ページは `.page-header`（sticky）+ `.content-area` の 2 層構造に揃える（Task 07〜10）
- 900px 以下でサイドバーがアイコンのみに縮小（CSS のみ、デザイン L1356〜1363）

### 3.5 テスト方針

- 既存テストは role / aria-* / テキストベース → **リスタイルで一切壊さないことが各タスクの完了条件**
- 新規挙動（テーマ切替・タブフィルタ・再生中強調・おすすめソース）は TDD（Red → Green → Refactor）で先行テスト
- 見た目そのもの（色・フォント）のユニットテストは書かない。クラス名の付与は `classList.contains` で最小限検証してよい（既存 ArticleCard テストの先例に倣う）

---

## 4. タスク構成と Wave（並列実行計画）

```
Wave 1（直列・基盤）   : T01
Wave 2（並列・最大9体）: T02 T03 T04 T05 T06 T07 T08 T09 T10
Wave 3（直列・統合）   : T11
```

| Task | 指示書 | 内容 | 占有ファイル（他タスク編集禁止） |
|---|---|---|---|
| T01 | `01-foundation.md` | globals.css 全量・フォント・app-shell・テーマ初期化 | `app/globals.css`, `app/layout.tsx`, `lib/config.ts`, `tests/app/layout.test.tsx` |
| T02 | `02-sidebar-theme.md` | サイドバー化・ロゴ・テーマトグル | `components/NavigationBar.tsx`, `components/ui/ThemeToggle.tsx`, `tests/components/NavigationBar.test.tsx`, `tests/components/ui/ThemeToggle.test.tsx` |
| T03 | `03-ui-parts.md` | Toast / ConfirmDialog / SetupModal / DifficultyBadge | `components/ui/{Toast,ConfirmDialog,SetupModal,DifficultyBadge}.tsx` + 対応テスト |
| T04 | `04-article-card.md` | ArticleCard リスタイル | `components/ArticleCard.tsx`, `tests/components/ArticleCard.test.tsx` |
| T05 | `05-podcast-card.md` | PodcastCard リスタイル + 再生中強調 | `components/PodcastCard.tsx`, `tests/components/PodcastCard.test.tsx` |
| T06 | `06-player-bar.md` | AudioPlayerBar リスタイル | `components/AudioPlayerBar.tsx`, `tests/components/AudioPlayerBar.test.tsx` |
| T07 | `07-feed-page.md` | /feed: ヘッダー・タブ・空状態 | `app/feed/page.tsx`, `tests/app/feed.test.tsx` |
| T08 | `08-podcast-pages.md` | /podcast 一覧 + 詳細 | `app/podcast/page.tsx`, `app/podcast/[id]/page.tsx` + 対応テスト |
| T09 | `09-subscriptions-page.md` | /subscriptions + おすすめソース | `app/subscriptions/page.tsx`, `tests/app/subscriptions.test.tsx` |
| T10 | `10-settings-entry.md` | /settings + / （エントリー） | `app/settings/page.tsx`, `app/page.tsx` + 対応テスト |
| T11 | `11-integration.md` | 統合検証・Docker 確認・シーム修正 | 全ファイル（ただし原則 globals.css の微調整のみ） |

**依存の根拠:** T02〜T10 はファイル集合が互いに素（disjoint）。コンポーネントの props 契約は不変（T05 の `playing?` 追加のみ後方互換）なので、ページタスクとコンポーネントタスクはファイル上独立に進められる。画面の見た目の整合は T11 で最終確認する。

### オーケストレーターの実行手順

1. `main` から `feature/webui-restyle` を作成（git-composer）
2. **Wave 1**: T01 の coder を起動。完了報告 → `npm run test` / `npm run build` 緑を確認 → git-composer がアトミックコミット（テスト/実装/構成は別コミット、日本語メッセージ、メタデータ禁止）
3. **Wave 2**: T02〜T10 の coder を**一斉に並列起動**（各 coder には該当指示書 1 枚 + 本書 §2・§3 を渡す）。全員完了後に `npm run test` 一括実行 → タスク単位で git-composer がコミット
4. **品質ゲート**: reviewer / ai-antipattern-reviewer を起動し PASS/REJECT 判定。REJECT は該当タスクの coder に差し戻し（指摘 + 元指示書を渡す）
5. **Wave 3**: T11 の coder を起動 → 統合検証・Docker 確認 → 最終レビュー → PR 作成（マージ判断は人間）

### coder 起動プロンプト（テンプレート・rule 91 準拠）

```markdown
あなたは coder です。`.claude/agents/coder.md` に従ってください。
taktステップ: execute_tdd
指示書: docs/plan/2026-06-12-webui-restyle/<NN-task>.md（全文を読むこと）
共通規約: docs/plan/2026-06-12-webui-restyle/00-overview.md の §2（D11〜D28）と §3 を遵守。
デザイン正本: docs/design/app-ui.html（指示書記載の行範囲を必ず参照）
制約:
- 指示書の「占有ファイル」以外を編集しない（globals.css 編集禁止。不足スタイルは申し送りに記載）
- 既存テストを 1 件も壊さない。新規挙動は Red → Green → Refactor
- git 操作禁止。完了時は coder.md の出力フォーマットで報告
```

---

## 5. 受け入れ基準（T11 完了時の検証項目）

1. `cd web && npm run test` — 全テストパス
2. `npm run build` — 型・lint エラーなし
3. `docker compose up --build api web` でローカル起動し、手動シナリオ:
   - 初回アクセス → SetupModal（新デザイン）→ 設定 → /feed がサイドバーレイアウトで表示
   - テーマトグルでダーク⇔ライト即時切替、リロード後も維持、`prefers-color-scheme` 初期反映
   - フィードタブ「すべて/★ スター済み」のフィルタと件数が動作。Star/Dismiss が従来どおり動く
   - /podcast で再生 → カードが amber 枠 + 波形で「再生中」表示、プレイヤーバーが新デザインで表示
   - /subscriptions のおすすめソースクリックでフォーム自動入力 → 追加成功
   - ウィンドウ幅 900px 以下でサイドバーがアイコンのみに縮小
4. 禁止フォント（Inter / Roboto / Arial / system-ui / Space Grotesk）が CSS・next/font 設定に存在しない
5. API キー・署名付き URL のログ出力なし。機能リグレッションなし（Star/Dismiss/再生位置復元/購読 CRUD/設定保存）

---

## 6. Phase 2 送り（本計画では着手禁止）

未読管理とフィードタブ「未読」/ 一括スター / Podcast 生成ステータス表示 / 相対日時表記 / キャッシュ管理 UI / 再生キュー（シャッフル）/ E2E テスト。これらに触れたくなった場合は実装せず、計画の更新を提案すること。
