# T01: デザイン基盤（globals.css・フォント・app-shell・テーマ初期化）

**Wave:** 1（単独実行。本タスク完了が T02〜T10 の起動条件）
**依存:** なし
**占有ファイル:** `web/app/globals.css`（新規）, `web/app/layout.tsx`, `web/lib/config.ts`, `web/tests/app/layout.test.tsx`
**参照:** `docs/design/app-ui.html` L10〜16（テーマ初期化スクリプト）, L17〜1363（CSS 全量）, L2001〜2003（spin keyframes）, L115〜120（app-shell グリッド）/ `00-overview.md` §3

## 目的

T02 以降の全タスクが「クラス名を付けるだけでデザインが当たる」状態を作る。CSS・フォント・テーマ・ルートレイアウトはこのタスクで完成させ、以降は凍結する。

## 実装内容

### 1. `app/globals.css`（新規作成）

- app-ui.html の `<style>` ブロック（L17〜1363）と `@keyframes spin`（L2001〜2003）を**全量移植**する。対象セクション: Reset & Tokens / Light Theme / Theme Transition / Noise overlay / Layout(app-shell) / Sidebar / Main Content / Page Header / Buttons / Content Area / Feed / Podcast / Subscriptions（add-card, form 含む）/ Settings / Audio Player Bar / Toast / Empty States / Theme Toggle / Scrollbar / Skeleton / Setup Modal / Responsive(@media 900px)
- 移植時の置換（これ以外は原文どおり。**独自の改変・省略をしない**）:
  - `font-family: 'Playfair Display', serif` → `font-family: var(--font-display), serif`
  - `font-family: 'DM Sans', sans-serif` → `font-family: var(--font-body), sans-serif`
  - `font-family: 'DM Mono', monospace` → `font-family: var(--font-mono), monospace`
  - `.app-shell` の `grid-template-rows: 1fr var(--player-h)` → `grid-template-rows: 1fr auto`（WHY: D26。プレイヤーバーは `currentPodcast` null 時に描画されないため、固定高だと空行が残る。`.player-bar` 自体に `height: var(--player-h)` を追加して描画時の高さを担保する）
  - L1460 付近のフィードタブ用インラインスタイル相当は `.feed-tabs` クラス（L349〜356）で代替できるため追加移植不要
- 既存ページがまだ旧マークアップである期間も崩壊しないよう、移植 CSS はクラスセレクタのみで構成されていることを確認する（要素型セレクタは `html, body` と scrollbar のみ — デザイン原文どおりで問題ない）

### 2. フォント（`app/layout.tsx`）

- `next/font/google` で `Playfair_Display`（weight 400/600/700/900）, `DM_Sans`（300/400/500/600）, `DM_Mono`（400/500）を読み込み、`variable: '--font-display' / '--font-body' / '--font-mono'` を指定
- 3 つの `.variable` クラスを `<body className>` に付与

### 3. テーマ初期化（`app/layout.tsx`）

- `<html lang="ja" suppressHydrationWarning>` とし、`<head>` 内に同期インラインスクリプト（`dangerouslySetInnerHTML`）を置く。内容は app-ui.html L10〜16 と同等:
  `localStorage.getItem('theme')` があればそれを、なければ `prefers-color-scheme` を `document.documentElement.dataset.theme` に設定
- WHY インライン: 外部スクリプトや useEffect では初回描画後に適用され、テーマが一瞬ちらつく（FOUC）

### 4. app-shell グリッド（`app/layout.tsx`）

- Provider 構成（AppProvider → ToastProvider → AudioPlayerProvider）は**変更しない**
- Provider 内を `<div className="app-shell">` で包み、子は `<NavigationBar />` → `<main className="main-content">{children}</main>` → `<AudioPlayerBar />` の 3 要素（NavigationBar / AudioPlayerBar の内部構造は T02 / T06 の責務。**コンポーネント自体は編集しない**）
- `globals.css` を import する

### 5. `lib/config.ts`

- localStorage キー定数 `KEY_THEME = 'theme'` を追加（T02 の ThemeToggle が使用）。既存定数・関数は変更しない

## TDD 手順（Red 観点）

`tests/app/layout.test.tsx`（既存があれば追記、なければ新規）:

1. RootLayout が children を描画する（既存挙動の維持）
2. `main` 要素（`role="main"`）が存在する
3. テーマ初期化スクリプトが `<head>` 相当の出力に含まれる（`localStorage` と `data-theme` を参照する script の存在をレンダリング結果の HTML 文字列で検証）

注意: RootLayout は `<html>` を返すため RTL の `render` でネスト警告が出る場合がある。既存テストの手法（あれば）に従い、なければ `renderToString` 等で HTML 文字列検証に切り替えてよい。

## 完了条件

- [ ] `npm run test` 全パス（既存テストを 1 件も壊さない）
- [ ] `npm run build` 成功（next/font のビルド時取得を含む）
- [ ] `npm run dev` でトップページにダークテーマ背景（`#0B0A0E`）とフォントが適用されている
- [ ] DevTools で `document.documentElement.dataset.theme` が `dark` または `light` に設定される

## 禁止事項

- `components/`・`contexts/`・`hooks/`・各ページの編集（占有ファイル外）
- CSS の独自アレンジ（色・寸法・アニメーションの変更）。正本は app-ui.html
- Tailwind ほか新規依存の追加（next/font は Next.js 同梱で依存追加なし）
