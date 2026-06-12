# T10: /settings ページ + /（エントリーページ）のリスタイル

**Wave:** 2（T01 完了後、他 Wave 2 タスクと並列実行可）
**依存:** T01（SetupModal の新デザイン適用は T03 の責務。本タスクは配置側のみ）
**占有ファイル:** `web/app/settings/page.tsx`, `web/app/page.tsx` + 対応テスト（`web/tests/app/` 配下の settings / page 関連）
**参照:** `docs/design/app-ui.html` L843〜945（Settings CSS）, L1836〜1935（マークアップ）/ `00-overview.md` §2 D14・D21・D22

## 目的

設定画面をデザインのセクションカード構造にする。**設定の保存ロジック（configure / useLocalStorage / 接続テスト）は不変**。

## 実装内容

### 1. `/settings`

- `.page-header`: `.page-title`「設定」+ `.page-subtitle`「アプリの動作をカスタマイズ」
- `.content-area`（`max-width: 600px` — L1845 のインラインスタイルどおり）に `.settings-section` を 2 つ:

**セクション 1「Podcast 生成」**（`.settings-section-icon` = 🎙️ / amber-dim 背景）
- `.settings-row`: 「デフォルト再生速度」ラベル + 説明。コントロールは**既存の 8 段階 `<select>` を維持**し `.select-input` クラスで装飾（D21。デザインの連続値スライダーは採用しない）
- 「デフォルト難易度」行は**セレクトを置かず**、`.settings-row-desc` で「Podcast の難易度はサーバー側設定で管理されています」を表示（D14・既存文言維持）

**セクション 2「API 接続設定」**（アイコン 🔌 / teal-glow 背景）
- baseUrl 入力（`.form-input`）・API キー（既存のマスク表示 + 再入力欄、`type="password"`）・接続テストボタン（`.btn btn-ghost`）— 挙動は既存のまま、構造を L1884〜1902 に揃える
- 注意書き（L1903〜1906 の「API キーは localStorage に保存されます…」）を追加してよい（静的テキスト）

- 保存ボタン: `.btn btn-primary` を右寄せ（L1928〜1932）。既存の保存フロー（configure 経由）不変
- **「ストレージ（キャッシュサイズ/クリア）」セクションは実装しない**（D22）

### 2. `/`（エントリーページ）

- 復元中スケルトン: `.skeleton` クラスでカード形状のプレースホルダーに（既存の表示条件 `isRestoring` は不変）
- 未設定時の SetupModal 表示・設定済み時の `/feed` リダイレクトは**ロジック不変**（見た目は T03 の SetupModal リスタイルで反映される）

## TDD 手順（Red 観点）

既存テスト（baseUrl 表示・キーのマスク・保存・速度セレクタ・接続テスト / エントリーのゲート挙動）を全てパスさせた上で追加:

1. settings: 「デフォルト難易度」のセレクト要素が存在しない（D14 のリグレッション防止）
2. settings: 速度セレクタが 8 段階のままである（`PLAYBACK_SPEEDS` と一致）
3. settings: 「キャッシュ」関連のボタンが存在しない（D22）

## 完了条件

- [x] `npm run test` 全パス
- [x] `npm run dev` で設定画面がセクションカード構造で表示され、保存・接続テストが従来どおり動く

## 禁止事項

- 難易度セレクト・キャッシュ管理の実装（D14/D22）
- `components/ui/SetupModal.tsx` の編集（T03 の占有）
- `contexts/AppContext.tsx`・保存ロジックの変更
- `globals.css` の編集
