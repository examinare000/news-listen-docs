# T08: /podcast 一覧・詳細ページのリスタイル

**Wave:** 2（T01 完了後、他 Wave 2 タスクと並列実行可）
**依存:** T01（T05 とはファイル独立。`playing` prop は省略可能なので T05 完了前でも型エラーにならない。ただし本タスクで `playing` を渡すコードを書くため、**Wave 2 内で T05 が型定義を先にコミットしていない場合は、`playing` の受け渡しテストが Red のまま T05 完了を待つ**。オーケストレーターは T05 → T08 の順で完了確認することが望ましい）
**占有ファイル:** `web/app/podcast/page.tsx`, `web/app/podcast/[id]/page.tsx` + 対応テスト（`web/tests/app/` 配下の podcast 関連）
**参照:** `docs/design/app-ui.html` L1613〜1747（Podcast ビュー）, L541〜545（グリッド CSS）/ `00-overview.md` §2 D11・D12・D24

## 目的

Podcast 一覧をデザインのカードグリッドにし、再生中エピソードの強調（D24）を結線する。詳細ページもページ構造を揃える。

## 実装内容

### 1. 一覧（`app/podcast/page.tsx`）

- `.page-header`: `.page-title`「ポッドキャスト」+ `.page-subtitle`「AI生成エピソード一覧」
- `.header-actions`: 更新ボタン（`.btn btn-icon`、既存の再取得ロジック）。**「新規生成」ボタンは実装しない**（D12）
- `.content-area` > `.podcast-grid` に `<PodcastCard>` を列挙
- **再生中強調の結線**: `useApp()` の `state.currentPodcast?.id === podcast.id` を `playing` prop として各カードに渡す（D24）
- ローディング: ページ内 SkeletonCard に `.skeleton` 適用（カード形状）
- 空状態: `.empty-state` 構造。文言は既存の「Podcast がまだありません。…」を維持

### 2. 詳細（`app/podcast/[id]/page.tsx`）

- `.page-header`: タイトル「エピソード詳細」+ 一覧へ戻るリンク（`.btn btn-ghost`）
- `.content-area`（`max-width` 制限はデザインの settings に倣い 600〜720px 程度。globals.css に専用クラスがなければインラインで `style={{maxWidth: ...}}` とし申し送りに記載）
- 本文: バッジ行（`<DifficultyBadge>` + digest 時 DIGEST）/ イントロ**全文**（既存）/ `.podcast-meta` 形式で時間・生成日 / 元記事 ID リスト（既存）/ 再生ボタン（`.btn btn-primary`、既存 useStartPodcast フロー）
- 404 時の「エピソードが見つかりません」+ 戻るリンクは `.empty-state` で表現（既存文言維持）

## TDD 手順（Red 観点）

既存テスト（一覧描画・空状態・再生フロー・404）を全てパスさせた上で追加:

1. 一覧: `currentPodcast` が podcast A のとき、A のカードにのみ「再生中」表示が出る（AppContext をテスト用 Provider で注入。既存テストの Provider ラップ手法に倣う）
2. 一覧: `currentPodcast` が null なら「再生中」がどこにも出ない
3. 詳細: 一覧へ戻るリンクが `/podcast` を指す

## 完了条件

- [x] `npm run test` 全パス
- [x] `npm run dev`（実 API）でグリッド表示・再生開始・再生中カードの強調（T05 結合後）が動く

## 禁止事項

- `components/PodcastCard.tsx` の編集（T05 の占有）
- `hooks/useStartPodcast.ts`・再生ロジックの変更
- ステータスバッジ・ポーリング・「新規生成」の実装（D11/D12）
- `globals.css` の編集
