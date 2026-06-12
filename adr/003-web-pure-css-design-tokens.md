# ADR-003: Web のスタイリングは純 CSS + globals.css 一元管理（Tailwind 不採用）

**ステータス:** 採用済み
**決定日:** 2026-06-12
**対象読者:** Web フロントエンド開発者

## 背景

WebUI リスタイル（PR #7）でデザイン正本が `docs/design/app-ui.html` に確定した。同ファイルは CSS 変数トークン・テーマ・全コンポーネントスタイルを含む完成された純 CSS であり、これをどの方式で実装へ移植するかの決定が必要だった。初期計画（2026-06-10）では Tailwind CSS v4 を想定していたが、実装には導入されていなかった。

## 検討した選択肢

| 選択肢 | メリット | デメリット |
|---|---|---|
| Tailwind CSS v4 を導入してユーティリティで再構築 | エコシステムが大きい | デザイン正本（純 CSS）からの翻訳作業が発生し移植忠実性の検証が困難。依存追加。既存テスト・ビルドへの影響 |
| CSS Modules / CSS-in-JS | スコープ分離 | 同上の翻訳コスト + ランタイム/ビルド複雑化 |
| **純 CSS を `app/globals.css` に全量移植し一元管理** | デザイン正本と 1:1 対応で diff 照合により忠実性を機械的に検証できる。依存ゼロ。コンポーネント側はクラス名付与のみ | グローバルスコープ（クラス名衝突は命名規約で回避） |

## 決定

`docs/design/app-ui.html` の `<style>` 全量を `web/app/globals.css` へ移植し、スタイルはこのファイルで一元管理する。コンポーネントはデザインのクラス名を付与するのみとし、コンポーネント単位の CSS ファイルや CSS-in-JS は作らない。フォントは `next/font/google`（Playfair Display / DM Sans / DM Mono）で CSS 変数（`--font-display` / `--font-body` / `--font-mono`）に紐付ける。

移植時の許容差分（これ以外は原文どおり）: フォントリテラルの変数化 / `.app-shell` の `grid-template-rows: 1fr auto` 化（プレイヤーバー非表示時の空行回避）/ `.player-bar` への `height` 追加 / 統合フェーズでの末尾追加（追加のみ・既存セレクタ無変更）。

## 理由

- デザイン正本が純 CSS の完成品であり、移植が最短かつ `diff` で忠実性を検証可能
- スタイルの単一ファイル占有により、複数 coder エージェントの並列実装でファイル競合をゼロにできる（リスタイルでは 9 タスク並列を無コンフリクトで完了）
- `next/font` のビルド時セルフホスティングで、コンテナ実行時の Google Fonts への外部リクエストを排除

## 結果（影響）

- 新しいスタイルが必要な場合は globals.css へ**追加のみ**行い、既存セレクタは変更しない（変更が必要ならデザイン正本の更新とセットで）
- デザイン上 `div` の操作要素も `<button>` / `<input type=range>` / `<select>` を維持し、アクセシビリティを劣化させない（速度ピルは `<select>` を `.speed-pill` 装飾、シークバーは range 入力）
- 禁止フォント（Inter / Roboto / Arial / system-ui / Space Grotesk — agent-rules/15）は CSS・next/font 設定に含めない
- Next.js build が自動生成する `tsconfig.json` 書き換えと `next-env.d.ts` は正式にコミットして追従する
