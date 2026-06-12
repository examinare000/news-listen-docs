# T09: /subscriptions ページのリスタイル + おすすめソース

**Wave:** 2（T01 完了後、他 Wave 2 タスクと並列実行可）
**依存:** T01
**占有ファイル:** `web/app/subscriptions/page.tsx`, `web/tests/app/subscriptions.test.tsx`
**参照:** `docs/design/app-ui.html` L696〜840（CSS）, L1749〜1834（マークアップ）/ `00-overview.md` §2 D13・D23

## 目的

購読管理を 2 カラムレイアウト（ソースリスト + sticky 追加カード）にし、新規挙動「おすすめのソース」（D23）を追加する。**API 連携（取得・追加・削除・エラーハンドリング・ConfirmDialog 経由の削除確認）のロジックは不変**。

## 実装内容

### 1. ページ構造

- `.page-header`: `.page-title`「購読管理」+ `.page-subtitle`「RSS ソースの追加・管理」
- `.content-area` > `.subs-layout`（2 カラムグリッド）:
  - 左: 件数表示「{N} ソース購読中」（`.subs-list` の上、L1761 のスタイル）+ `.subs-list`
  - 右: `.add-card`（sticky）

### 2. ソース行（`.sub-row`）

- `.sub-icon`（絵文字 1 つ。ソース毎の出し分けはせず固定で「📡」。WHY: デザインの 🔺🟢 はモック用の手動割当で、URL からの推定は本計画のスコープ外）
- `.sub-info` > `.sub-name` + `.sub-url`
- `.sub-delete`: ゴミ箱 SVG（L1772）。既存の ConfirmDialog → `deleteSource(url)` フローに接続。アクセシブルネーム「削除」維持
- **有効/無効トグル（`.sub-toggle`）は実装しない**（D13）

### 3. 追加カード（`.add-card`）

- `.add-card-title`「ソースを追加」+ `.add-card-desc`
- 既存フォーム（name / url 入力・クライアント側 URL 検証・409/422 エラー表示・送信中 disabled）を `.form-field` / `.form-label` / `.form-input` 構造に。送信ボタンは `.btn btn-primary` 全幅
- **おすすめのソース（新規挙動・TDD 必須）**: カード下部に区切り + 「おすすめのソース」見出し + 2 件（The Verge `https://www.theverge.com/rss/index.xml` / dev.to `https://dev.to/feed`、L1814〜1827）。各「追加」ボタン（`.btn btn-ghost`）クリックで**フォームの name/url に値を入力し name 入力欄へフォーカス**する（即 API 送信はしない。WHY: ユーザーが URL を確認してから登録する余地を残す = デザインの `fillForm` と同挙動）

### 4. 状態表示

- 空状態: `.empty-state` 構造で既存文言「購読ソースがありません」+ 追加フォームへの誘導

## TDD 手順（Red 観点）

既存テスト（一覧・追加・削除・409/422/404・空状態）を全てパスさせた上で追加:

1. おすすめソース「The Verge」の追加ボタンクリック → name 入力欄に「The Verge」、url 入力欄に該当 URL が入る
2. クリック後に name 入力欄がフォーカスされている（`document.activeElement` 検証）
3. おすすめクリックだけでは `addSource` が呼ばれない（fetch モックが未呼出）
4. 「{N} ソース購読中」が一覧件数と一致し、削除成功後に減る

## 完了条件

- [ ] `npm run test` 全パス
- [ ] `npm run dev`（実 API）で 2 カラム表示・追加・削除・おすすめ入力が動く

## 禁止事項

- 有効/無効トグルの実装（D13）
- `lib/api.ts`・ConfirmDialog（T03 占有）の編集
- おすすめソースの即時 API 登録化（フォーム入力までが仕様）
- `globals.css` の編集
