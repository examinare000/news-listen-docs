# T04: ArticleCard のリスタイル

**Wave:** 2（T01 完了後、他 Wave 2 タスクと並列実行可）
**依存:** T01
**占有ファイル:** `web/components/ArticleCard.tsx`, `web/tests/components/ArticleCard.test.tsx`
**参照:** `docs/design/app-ui.html` L381〜538（CSS）, L1470〜1607（マークアップ実例 5 件）/ `00-overview.md` §2 D28

## 目的

フィードの中核部品をデザインのカード（左帯ホバー・スコアバー・丸型アクションボタン）に置き換える。**props 契約 `{ article, onStar, onDismiss, busy, starred }` は不変**（/feed ページタスク T07 と並列のため）。

## 実装内容

デザインのマークアップ（L1470〜1495 が代表例）を React 化する:

- ルート: `<div className="article-card">`、`starred` が真なら `article-card starred`
- 左カラム:
  - `.article-meta`: `.article-source`（source 名）+ `.article-dot`「·」+ `.article-date`（既存 `formatDate` 出力のまま。相対表記化はしない = D28）
  - `.article-title` > `<a href={article.url} target="_blank" rel="noopener noreferrer">`（既存の外部リンク属性を維持）
  - `.score-row`: `.score-bar-track` > `.score-bar-fill`（`width: ${score*100}%` のインラインスタイル）+ `.score-label`「{score.toFixed(2)} 関連度」。既存の `role="progressbar"` / `aria-valuenow` は track 側に維持
- 右カラム `.article-actions`:
  - スター: `.action-btn star`（active 時 `action-btn star active`）。SVG はデザイン L1489（塗り）/ L1517（線）を状態で切替。既存の `aria-pressed`・アクセシブルネーム（スター）・`disabled`（busy 時）を維持。**`<button>` 要素を維持する**（デザインは div だが a11y 劣化させない）
  - 非表示: `.action-btn dismiss`。SVG は L1492。同様に `<button>` + アクセシブルネーム維持

## TDD 手順（Red 観点）

既存テスト（role/aria/テキストベース + `classList.contains('starred')`）を全てパスさせた上で追加:

1. `starred=true` でルートカードに `starred` クラスが付く（既存テストがボタン側を見ている場合は両方検証）
2. スコアバーの fill 要素の `style.width` が `score * 100%` と一致（score 0 / 1.0 の境界値）
3. source・「関連度」ラベルが表示される

## 完了条件

- [ ] `npm run test` 全パス
- [ ] `npm run dev` の /feed でカードがデザイン外観（ホバーで左帯 amber、スター点灯で amber 塗り）になる

## 禁止事項

- props 契約の変更、API 呼び出しの追加（カードは純粋な制御コンポーネントのまま）
- `app/feed/page.tsx` の編集（T07 の占有）
- `lib/format.ts` の編集（D28）・`globals.css` の編集
