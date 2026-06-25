# ADR-031: Web 主要動線 E2E（Playwright）とグローバルエラーバウンダリ

**ステータス:** 採用済み
**決定日:** 2026-06-25
**対象読者:** Web フロントエンド開発者
**関連:** [web-design.md](../design/web-design.md)・`web/playwright.config.ts`・`web/e2e/`・`web/app/error.tsx`・`web/app/global-error.tsx`・PRD §13（P2）・issue #48

## 背景

Web は vitest のユニット／コンポーネントテストのみで、ブラウザ実機レベルの主要動線（ログイン→フィード→Star→再生）を保証する E2E が無かった。また App Router の `error.tsx`／グローバルエラーバウンダリが無く、予期しない実行時例外で白画面化するリスクがあった（PRD §13 P2 の品質・堅牢性課題）。

## 検討した選択肢

| 論点 | 選択肢 | 評価 |
|---|---|---|
| E2E ランナー | (a) Playwright（採用） | App Router・`page.route` スタブ・ヘッドレス Chromium に対応。標準的 |
| | (b) Cypress | 同等だが既存スタックとの親和性で Playwright を優先 |
| E2E のバックエンド | (a) `page.route('**/api/backend/**')` で BFF をスタブ（採用） | ネットワーク非依存・決定的。CI に実バックエンド不要 |
| | (b) 実バックエンドを起動 | 重く不安定。主要動線の回帰検出には過剰 |
| webServer | (a) 本番ビルド（`next build && next start`、採用） | `error.tsx` が `next dev` のエラーオーバーレイに隠れず実挙動を検証できる |
| | (b) `next dev` | HMR/オーバーレイで非決定的。エラー境界を検証できない |
| エラー境界の検証 | (a) vitest コンポーネントテスト（採用） | 決定的・高速。no-leak 契約を構造的に固定 |
| | (b) 本番に意図的 throw ルートを同梱 | 本番へ throw ページを出荷するのは不適切。不採用 |

## 決定

- **Playwright を導入**（`@playwright/test`、chromium のみ）。`e2e/` に配置し `testMatch: '**/*.e2e.ts'`。vitest 側は `exclude: ['e2e/**']` を追加し、両ランナーがテストを取り違えないよう二重防御する。
- **主要動線 E2E** `e2e/main-flow.e2e.ts`: localStorage に接続設定をシード→ログイン→`/feed` で記事表示→Star（成功トースト／スター状態）→`/podcast` へ遷移→再生（`AudioPlayerBar` の再生状態を確認）。API は同一オリジン `**/api/backend/**` を `page.route` でスタブ。**キャッチオールは最初に登録**する（Playwright は後勝ちマッチのため、後続の個別スタブが優先される）。音声は再生可能な無音 WAV を返し、`audio.play()` の resolve 後に `SET_PODCAST` がディスパッチされる経路を検証する。
- **webServer は本番ビルド**（`next start`）で起動する。`next dev` のエラーオーバーレイが `error.tsx` を覆い隠すため。
- **エラーバウンダリ**: `app/error.tsx`（ルートセグメント。既存 `empty-state` トークンで安全フォールバック＋`reset()`）と `app/global-error.tsx`（root `layout.tsx` の例外専用。自前の `<html><body>` を描画）を追加。いずれも `error.message`／`error.stack`／`error.digest` を UI・ログに出さない。安全フォールバックと no-leak 契約は `tests/app/error.test.tsx` で固定する。

## 理由

- BFF スタブ＋本番ビルドにより、実バックエンド無しで主要動線とエラー境界の実挙動を決定的に検証できる。
- エラー境界の受け入れ条件（安全フォールバック・内部情報の非漏洩）は vitest で決定的に固定でき、本番へ throw ルートを出荷せずに済む。
- ランナー隔離を exclude と命名規約の双方で担保し、ユニットテストへの E2E 混入を構造的に防ぐ。

## 結果（影響）

- 追加: `playwright.config.ts`・`e2e/main-flow.e2e.ts`・`e2e/fixtures/silence.wav`・`app/error.tsx`・`app/global-error.tsx`・`tests/app/error.test.tsx`。`package.json` に `test:e2e`／`test:e2e:install`、`vitest.config.ts` に exclude、`.gitignore` に Playwright 生成物。E2E 用に `ArticleCard`／`PodcastCard`／`AudioPlayerBar` へ `data-testid` を付与（挙動変更なし）。
- 受け入れ条件: 主要動線 E2E が green、予期しない例外で安全な日本語フォールバックを表示し内部情報を漏らさない。

## トレードオフと留保事項

- **CI 未整備**: 本リポジトリに GitHub Actions 等の CI ワークフローが存在しないため、E2E は当面ローカル（`npm run test:e2e`）で green を担保する。CI 導入時に `test:e2e:install`→`test:e2e` のジョブを追加する（chromium のみ）。
- **本番ビルドで実行するため E2E は遅い**（ビルドコスト）。決定性とエラー境界検証を優先したトレードオフ。
- **動線は `/feed`→`/podcast` をまたぐ**: Star は `/feed`、再生は `/podcast` という実アーキテクチャに合わせている。
