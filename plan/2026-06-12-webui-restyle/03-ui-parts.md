# T03: 汎用 UI 部品のリスタイル（Toast / ConfirmDialog / SetupModal / DifficultyBadge）

**Wave:** 2（T01 完了後、他 Wave 2 タスクと並列実行可）
**依存:** T01
**占有ファイル:** `web/components/ui/Toast.tsx`, `web/components/ui/ConfirmDialog.tsx`, `web/components/ui/SetupModal.tsx`, `web/components/ui/DifficultyBadge.tsx` + `web/tests/components/ui/` 配下の対応テスト 4 ファイル
**参照:** `docs/design/app-ui.html` L1146〜1187（Toast CSS）, L2092〜2111（Toast 挙動）, L1300〜1348（Modal CSS）, L598〜616（badge CSS）/ `00-overview.md` §2 D27

## 目的

全画面から使われる共通部品にデザインのクラス・マークアップを適用する。**インターフェース（props・Context API・文言）は一切変更しない**（並列実行中の他タスクがこれらを参照しているため）。

## 実装内容

### 1. Toast

- コンテナに `.toast-container`、各トーストに `.toast toast-success` / `.toast toast-error`（error はデザインの `toast-info` 配色定義しかないため、`toast-error` クラスを付与しつつ見た目は申し送りに記載。アイコンは success=`✓` / error=`!`）
- `.toast-icon` 要素を追加。`role="status"` / `aria-live` 等の既存アクセシビリティ属性・3 秒自動消滅・`useToast()` API は不変
- Star 時の文言は呼び出し側の責務（D27）。本タスクでは触らない

### 2. ConfirmDialog

- バックドロップに `.modal-backdrop`、本体に `.modal-box`。タイトルは `.modal-title`、本文は `.modal-desc`
- ボタン: 確認 = `.btn btn-primary`、キャンセル = `.btn btn-ghost`
- `role="dialog"` / `aria-modal` / Escape クローズ / コールバック契約は不変

### 3. SetupModal

- `.modal-backdrop` + `.modal-box` 構造に変更し、ヘッダーに `.modal-logo`（NavigationBar と同じロゴ SVG + ロゴテキスト。SVG はデザイン L1375〜1378 からコピー）、`.modal-title`「ようこそ」相当 + `.modal-desc`（既存の説明文言を維持）
- 入力欄を `.form-field` > `.form-label` + `.form-input` 構造に。既存の `id`・`type="password"`・バリデーション・接続テストボタン（`.btn btn-ghost`）・保存ボタン（`.btn btn-primary`）の挙動は不変

### 4. DifficultyBadge

- `<span className="badge badge-{tone}">` に変更。難易度→トーンの対応（デザイン L614〜616 の 3 色を 6 難易度に割当）:
  - `toeic_600`・`eiken_2` → `badge-easy`（green）
  - `toeic_900`・`ielts_55` → `badge-medium`（amber）
  - `ielts_7`・`eiken_p1` → `badge-hard`（purple）
  - 未知の値 → トーンクラスなしの `badge` のみ + 生値表示（既存の境界値挙動を維持）
- 表示ラベル（「TOEIC 900」等）は既存マッピングを変更しない

## TDD 手順（Red 観点）

既存テストはすべてそのままパスさせること。追加する Red:

1. DifficultyBadge: 6 難易度それぞれで期待トーンのクラス（`badge-easy` 等）が付与される / 未知の値でトーンクラスが付かず生値が表示される
2. Toast: success と error で異なるクラスが付与される（`classList.contains` で検証）
3. ConfirmDialog / SetupModal: 既存の role・挙動テストがパスし続けることを確認（マークアップ変更でクエリが壊れた場合、**クエリを role/text ベースに直す方向で**修正する。挙動の仕様変更はしない）

## 完了条件

- [ ] `npm run test` 全パス
- [ ] `npm run dev` で SetupModal（localStorage クリアで再現）がデザインのモーダル外観で表示される
- [ ] Toast がカード型 + アイコン付きで右上に表示される

## 禁止事項

- props・Context のシグネチャ変更、文言変更（D27）
- `globals.css` の編集（`toast-error` 配色の不足等は申し送りへ）
- SkeletonCard の共通コンポーネント化（各ページ内定義のまま。ページタスクとの競合防止）
