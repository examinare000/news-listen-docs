# T07: /feed ページのリスタイル + タブフィルタ

**Wave:** 2（T01 完了後、他 Wave 2 タスクと並列実行可）
**依存:** T01（T04 とはファイル独立。ArticleCard の props 契約は不変なので結合は自動的に成立する）
**占有ファイル:** `web/app/feed/page.tsx`, `web/tests/app/feed.test.tsx`
**参照:** `docs/design/app-ui.html` L1443〜1610（フィードビュー全体）, L348〜378（タブ CSS）, L1189〜1209（空状態）/ `00-overview.md` §2 D15・D16・D27・D28

## 目的

フィード画面をデザインのページ構造（sticky ヘッダー + タブ + カードリスト）にし、新規挙動としてタブによるクライアント側フィルタを追加する。**API 呼び出し・Star/Dismiss・トースト文言（D27）のロジックは不変**。

## 実装内容

### 1. ページ構造

- `.page-header`: `.page-title`「フィード」+ `.page-subtitle`「今日のレコメンド記事 — {feed.date}」（API の `date` を使用。未取得時は省略）
- ヘッダー右 `.header-actions`: 更新ボタン（`.btn btn-icon`、リフレッシュ SVG、既存の再取得ロジックに接続、アクセシブルネーム「更新」維持）。**「全部スターする」は実装しない**（D16）
- タブ行（L1460〜1464 の構造を `.feed-tabs` で実装）
- `.content-area` > `.article-list` に `<ArticleCard>` を列挙（props 契約は現行どおり）

### 2. タブフィルタ（新規挙動・TDD 必須）

- タブは 2 つ: 「すべて {全件数}」「★ スター済み {starredIds 件数}」（D15。「未読」は実装しない）
- 選択タブを `useState` で保持。「スター済み」選択時は `starredIds` に含まれる記事のみ表示
- タブはキーボード操作可能な要素（`<button>` か `role="tab"`）とし、選択状態を `aria-selected` または `aria-pressed` で表現
- Dismiss で記事が消えた場合も件数表示が追従する（既存 state から導出するだけにする）

### 3. 状態表示

- ローディング: 既存の SkeletonCard（ページ内定義のまま）に `.skeleton` クラスを適用し、カード形状（高さ・角丸）のプレースホルダーへ
- 空状態: `.empty-state`（アイコン `.empty-state-icon` + `.empty-state-title` + `.empty-state-desc`）。文言は既存の「まだ記事がありません。…」を維持。スター済みタブで 0 件の場合は「スター済みの記事はありません」

## TDD 手順（Red 観点）

既存テスト（取得・Star/Dismiss・トースト・空状態・リフレッシュ）を全てパスさせた上で追加:

1. タブ「★ スター済み」クリック → スター済み記事のみ表示、未スター記事が消える
2. 「すべて」に戻すと全件表示
3. 各タブに件数が表示され、Star 操作後に「スター済み」の件数が増える
4. スター済みタブで 0 件のとき専用の空状態文言が出る

## 完了条件

- [x] `npm run test` 全パス
- [x] `npm run dev`（実 API）でヘッダー・タブ・カードリストがデザイン外観で表示され、Star/Dismiss/更新が従来どおり動く

## 禁止事項

- `components/ArticleCard.tsx` の編集（T04 の占有）。props 契約の変更要求が生じたら申し送りへ
- 楽観更新の導入・API 層の変更
- 「未読」タブ・一括スターの実装（D15/D16）
- `globals.css` の編集
