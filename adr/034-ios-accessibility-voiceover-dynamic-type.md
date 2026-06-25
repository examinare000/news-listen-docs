# ADR-034: iOS アクセシビリティ（VoiceOver ラベル・Dynamic Type 対応）

**ステータス:** 採用済み
**決定日:** 2026-06-26
**対象読者:** iOS 開発者
**関連:** [ADR-010](010-ios-architecture-swiftui-mvvm.md)・[ADR-032](032-web-modal-focus-trap.md)（web 側アクセシビリティ）・PRD §13（P2・包摂性）・issue #49

## 背景

issue #49 の iOS 部分。score バーには一部 `accessibilityLabel`/`accessibilityValue` があったが、それ以外の主要操作（記事行・Podcast 行・各種ボタン・オーディオプレイヤー）に VoiceOver ラベルが無く、`AudioPlayerView` の再生アイコンが `.font(.system(size: 52))` の固定サイズで Dynamic Type に追従しなかった。受け入れ条件は「VoiceOver/キーボードのみで主要動線が操作可能」。

## 決定

- **VoiceOver ラベル/ヒント**を主要操作に付与する。
  - `ArticleRowView`: 行を `.accessibilityElement(children: .combine)` で 1 要素にまとめ、`accessibilityLabel`（記事タイトル）・`accessibilityValue`（ソース・公開日）・`accessibilityHint`（タップで開く／左右スワイプでスター・削除）を設定。既存の score バー a11y は維持。
  - `FeedView`: 更新・選択モードトグル・一括スター/キャンセル・スワイプ（Star/Dismiss）の各ボタンに状態依存ラベルを付与。
  - `PodcastRowView`/`PodcastView`: 行をまとめてラベル＋ヒント、ダウンロードボタンを状態別ラベル化。
  - `AudioPlayerView`: 15秒戻す／再生・一時停止（状態依存）／30秒進む にラベル・ヒント。
  - `SettingsView`: 「記事の開き方」「記事の日付表記」の Picker にヒント。
- **Dynamic Type**: `AudioPlayerView` の再生アイコンを `.font(.system(size: 52))` から `.font(.system(.largeTitle))`（テキストスタイル指定）へ変更し、ユーザーの文字サイズ設定に追従させる。`.caption`/`.headline`/`.subheadline`/`.title2` 等の既存テキストスタイルは元から Dynamic Type 対応のため変更不要。

## 理由

- SwiftUI の標準アクセシビリティ修飾子で完結し、依存追加なし。テキストスタイル指定により固定サイズの追従不全を解消する。
- 行を 1 要素にまとめることで VoiceOver のフォーカス移動が冗長にならず、主要操作（開く・スター・削除・再生）が明確に読み上げられる。

## 結果（影響）

- 変更: `Feed/ArticleRowView.swift`・`Feed/FeedView.swift`・`Podcast/PodcastRowView.swift`・`Podcast/PodcastView.swift`・`Podcast/AudioPlayerView.swift`・`Settings/SettingsView.swift`。
- 既存の XCTest 96 件は引き続き green（アクセシビリティ修飾子はテスト数に影響しない）。

## トレードオフと留保事項

- **VoiceOver の実読み上げ・Dynamic Type の大文字サイズでのレイアウト崩れは自動テストで検証できない**。Xcode/Simulator で VoiceOver 有効化・文字サイズ拡大での手動確認に委ねる（残課題）。
- 行を `.combine` でまとめたため score バーの百分率は単独要素としては読み上げられない（主要操作の操作性を優先した割り切り）。
- web 側のアクセシビリティ（モーダルのフォーカストラップ）は [ADR-032](032-web-modal-focus-trap.md) を参照。
