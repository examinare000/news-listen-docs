# ADR-032: Web モーダルのフォーカストラップ・フォーカス復帰

**ステータス:** 採用済み
**決定日:** 2026-06-25
**対象読者:** Web フロントエンド開発者
**関連:** [web-design.md](../design/web-design.md)・`web/hooks/useFocusTrap.ts`・`web/hooks/focusableElements.ts`・PRD §13（P2・アクセシビリティ）・issue #49

## 背景

モーダル（SetupModal / LoginModal / OnboardingSourcesModal / ConfirmDialog）はいずれも `role="dialog"`・`aria-modal`・`aria-labelledby` を備えていたが、フォーカストラップ・初期フォーカス・閉じた後のフォーカス復帰が無く、キーボード／スクリーンリーダー利用者がモーダル外へ簡単にフォーカスを逃がせてしまう状態だった（PRD §13 P2 のアクセシビリティ課題）。受け入れ条件は「キーボードのみで主要動線が操作可能」。

## 検討した選択肢

| 論点 | 選択肢 | 評価 |
|---|---|---|
| 実装方式 | (a) 自前の再利用フック `useFocusTrap`（採用） | 依存追加なし。挙動を完全に制御・テスト可能 |
| | (b) focus-trap 等の外部ライブラリ | 依存追加に見合うほど複雑ではない。YAGNI |
| 構造 | (a) 純関数 `getFocusableElements` ＋ 副作用シェル `useFocusTrap` に分離（採用） | 最も回帰しやすい列挙ロジックをフレームワーク非依存で単体テストできる |
| | (b) 単一フックに同居 | 列挙ロジックの単体テストが難しい |
| 可視判定 | (a) 属性ベース（`disabled`/`hidden`/`aria-hidden`/`tabindex="-1"`）（採用） | jsdom はレイアウト未計算で `offsetParent` が常に null になるためテスト不能 |
| | (b) `offsetParent` で可視判定 | jsdom で誤判定。不採用 |
| Escape | (a) フックでは扱わず各モーダルの責務（採用） | 閉じる意味づけはモーダルごとに異なる。ConfirmDialog は既存 Escape を維持、他3つは設計上 Escape で閉じない |

## 決定

- **`hooks/focusableElements.ts`**: 純関数 `getFocusableElements(container)`。`a[href]`・`button/input/select/textarea:not([disabled])`・`[tabindex]:not([tabindex="-1"])` を対象に、`hidden`/`aria-hidden`/`disabled` を除外（属性ベース）。
- **`hooks/useFocusTrap.ts`**（`'use client'`）: `const ref = useFocusTrap<T>(isOpen?)` を返し、ダイアログコンテナ（`role="dialog"` の要素）へ付与する。
  - 開いたとき: `document.activeElement` を復帰先として保存し、最初の focusable（無ければコンテナ自身に一時 `tabindex=-1`）へフォーカス。
  - Tab/Shift+Tab: ダイアログ内の最初/最後で循環させる（外へ逃がさない）。
  - 閉じる/アンマウント時: 起動元へフォーカス復帰。**ただし起動元がまだ DOM 接続済み（`isConnected`）の場合のみ**——ConfirmDialog の削除確定では起動元（行の削除ボタン）が確定後に外れるため、detach 済み要素への `focus()`（無音失敗で focus が body へ飛ぶ）を避ける。
- **適用**: マウント＝オープンの3モーダル（Setup/Login/Onboarding）は引数なしで呼ぶ。常時マウントで `isOpen` 切替の ConfirmDialog は `isOpen` を明示的に渡す。既存の aria 属性・Escape 挙動は変更しない。

## 理由

- 列挙ロジックを純関数に隔離することで、focusable 判定（種別・disabled・hidden・tabindex）を I/O なしで網羅テストでき、フォーカストラップ本体は `fireEvent.keyDown` で決定的に検証できる。
- 属性ベース可視判定により jsdom 上でもテストが成立する。
- `isConnected` ガードにより、削除フローでフォーカスが宙に浮く事故を構造的に防ぐ。

## 結果（影響）

- 追加: `hooks/focusableElements.ts`・`hooks/useFocusTrap.ts` と各テスト。4モーダルへフックを適用。
- 受け入れ条件: 初期フォーカス・Tab 循環・閉じた後の復帰（起動元 detach 時は非復帰）を vitest で検証。549 件のテストが green。

## トレードオフと留保事項

- **jsdom の限界**: 実ブラウザの Tab 視覚移動・スクリーンリーダー（VoiceOver 等）の読み上げは自動テストでは検証できず、手動確認に委ねる。
- **iOS の VoiceOver/Dynamic Type は別対応**（issue #49 iOS 部分）。
