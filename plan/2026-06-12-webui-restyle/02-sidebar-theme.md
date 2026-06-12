# T02: サイドバー化とテーマトグル

**Wave:** 2（T01 完了後、T03〜T10 と並列実行可）
**依存:** T01（globals.css のクラス・`KEY_THEME` 定数が存在すること）
**占有ファイル:** `web/components/NavigationBar.tsx`, `web/components/ui/ThemeToggle.tsx`（新規）, `web/tests/components/NavigationBar.test.tsx`, `web/tests/components/ui/ThemeToggle.test.tsx`（新規）
**参照:** `docs/design/app-ui.html` L1371〜1437（サイドバーのマークアップ）, L1425〜1435（テーマトグル）, L2007〜2012（toggleTheme の挙動）/ `00-overview.md` §2 D17・D18・D25, §3.3

## 目的

上部テキストリンクのみの NavigationBar を、デザインのサイドバー（ロゴ + アイコン付きナビ + フッターのテーマトグル）に置き換える。

## 実装内容

### 1. `NavigationBar.tsx` → サイドバー化

- ルート要素を `<aside className="sidebar">` とし、内部に `<nav aria-label="メインナビゲーション">` を保持（既存テストの aria-label を維持）
- 構成（デザイン L1371〜1437 のマークアップを React 化）:
  - `.sidebar-logo` > `.logo-mark`: ロゴアイコン SVG（L1375〜1378 をコピー）+ `.logo-text`「Audio**News**」（News 部分を `<span>` で amber）
  - `.nav-section-label`「メニュー」
  - ナビ 4 項目: フィード `/feed` / ポッドキャスト `/podcast` / 購読管理 `/subscriptions` / 設定 `/settings`。各項目はデザインの SVG アイコン（L1388〜1414）+ 日本語ラベル `<span>`
  - 各項目は Next.js `<Link>` のまま、`className="nav-item"`、現在パスで `nav-item active` + `aria-current="page"`（既存ロジック維持）
  - 設定の前に `.sidebar-divider`
  - `.sidebar-footer` に `<ThemeToggle />` のみ配置
- **実装しないもの:** ナビバッジ（未読数）= D17、「バッチ: 正常稼働中」ステータス = D18
- リンクテキストは日本語化する（フィード/ポッドキャスト/購読管理/設定）。既存テストが英語名（Feed 等）でクエリしている場合は**テストも日本語に更新**してよい（表示文言の変更はデザイン反映の一部）

### 2. `components/ui/ThemeToggle.tsx`（新規）

- デザイン L1425〜1435 のマークアップ（`.theme-toggle` ボタン + track/thumb + 🌙/☀️ ラベル）を React 化。`<button type="button">` で `aria-label="テーマ切替"` を付与
- クリック時: `document.documentElement.dataset.theme` を `dark ⇄ light` でトグルし、`localStorage` の `KEY_THEME`（`lib/config.ts` から import）へ**生文字列で**保存（注意: 既存 `useLocalStorage` は JSON.stringify するため、T01 の初期化スクリプトが読む生値と互換にならない。このコンポーネントでは `localStorage.setItem` を直接使う。WHY を実装コメントに残すこと）
- React state は持たない。表示の dark/light 切替は CSS（`html[data-theme]` セレクタ）が担う

### 3. アイコンの扱い

- SVG はデザイン HTML から属性を JSX 形式（`strokeWidth` 等）に変換してインライン埋め込み。アイコンライブラリは導入しない

## TDD 手順（Red 観点）

1. **NavigationBar（既存テスト更新 + 追加）:**
   - 4 リンクが描画され、各 `href` が正しい
   - 現在パスのリンクに `aria-current="page"`（既存観点の維持）
   - ルートが `complementary` ロール（aside）として描画される
2. **ThemeToggle（新規）:**
   - 初期 `data-theme="dark"` でクリック → `document.documentElement.dataset.theme === 'light'` になり、`localStorage.getItem('theme') === 'light'`
   - もう一度クリック → `dark` に戻る
   - `role="button"` とアクセシブルネームを持つ
   - テストの setup/teardown で `document.documentElement.dataset.theme` と localStorage を初期化すること

## 完了条件

- [ ] `npm run test` 全パス
- [ ] `npm run dev` でサイドバーがロゴ・アイコン付きで左固定表示され、テーマトグルで全体の配色が即時切替・リロード後も維持される
- [ ] 幅 900px 以下でラベルが消えアイコンのみになる（CSS は T01 で導入済み。マークアップがメディアクエリのセレクタと一致していること）

## 禁止事項

- `app/layout.tsx`・`globals.css` の編集（不足スタイルは申し送りへ）
- テーマ状態の Context / AppContext への追加（§3.3: DOM が単一の真実）
- アイコンライブラリ等の依存追加
