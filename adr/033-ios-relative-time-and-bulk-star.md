# ADR-033: iOS 記事の相対日時表記（切替可）と複数選択一括スター

**ステータス:** 採用済み
**決定日:** 2026-06-25
**対象読者:** iOS 開発者
**関連:** [ADR-030](030-web-relative-time-and-bulk-star.md)（web 版）・[ADR-008](008-ios-local-state-persistence.md)・[ADR-010](010-ios-architecture-swiftui-mvvm.md)・`ios/NewsListenApp/NewsListenApp/Utilities/RelativeTimeFormatter.swift`・PRD §13（P2）・issue #50

## 背景

issue #50 の iOS 部分。記事/Podcast の日付は `publishedAt.prefix(10)`（`YYYY-MM-DD` の絶対表記）固定で、相対表記（「3時間前」）と絶対/相対の切替が無かった。スターは行スワイプの 1 件操作のみで、複数記事をまとめて star する手段が無かった。web は [ADR-030](030-web-relative-time-and-bulk-star.md) で実装済みのため、**同等の挙動を iOS にも提供する**。

## 決定

- **相対日時**: `RelativeTimeFormatter.format(_ isoString: String, now: Date = Date()) -> String` を追加（純構造体・`now` 注入で決定的テスト）。意味論は ADR-030 と一致: パース失敗→`""`、未来→`もうすぐ`、<60秒→`たった今`、分/時/日→`N分前`/`N時間前`/`N日前`、月（30日単位）→`Nか月前`、年（365日単位）→`N年前`。**年を月より先に判定**し、360–364 日で `0年前` にならないようにする。
- **表示設定**: `AppState.timeFormat`（`"absolute"`/`"relative"`、既定 `"absolute"`）を追加。UserDefaults キー `time_format`（[ADR-008](008-ios-local-state-persistence.md) の `articleOpenMode` と同じ didSet 永続化パターン）。`SettingsView` にセレクト（Picker）を追加。`ArticleRowView`/`PodcastRowView` は `appState.timeFormat` に応じて相対/絶対を描画。
- **一括スター**: `FeedViewModel` に `isSelectionMode`・`selectedIds`・`toggleSelection(_:)`・`bulkStar()` を追加。`bulkStar()` は `withTaskGroup` で並行に `starArticle(id:)` を呼び、部分失敗を許容して成功/失敗件数を `BulkActionResult` に集計、成功分を一覧から除去する。**現在表示中の記事に限定**（`selectedIds ∩ articles`）してリフレッシュ後の消えた記事を star しない。`FeedView` は選択モードのトグル・行チェックマーク・「N件を一括スター」ボタンを提供し、選択モード中はスワイプ操作を無効化して個別操作と排他する。

## 理由

- web ADR-030 と意味論を揃えることでクロスプラットフォームの体験差を最小化する。`now` 注入と純構造体化により相対表記の境界を XCTest で網羅検証できる。
- `withTaskGroup` ＋ 部分失敗許容と「現在表示中限定」により、ネットワーク部分失敗・リフレッシュ競合に堅牢に振る舞う。
- 永続化は既存の UserDefaults パターン（ADR-008）に従い、新たな仕組みを増やさない。

## 結果（影響）

- 追加: `Utilities/RelativeTimeFormatter.swift` と各 XCTest。変更: `AppState.swift`（`timeFormat`）・`Settings/SettingsView.swift`・`Feed/ArticleRowView.swift`・`Podcast/PodcastRowView.swift`・`Feed/FeedViewModel.swift`・`Feed/FeedView.swift`。
- 受け入れ条件: 相対表記の各境界（未来/たった今/分/時/日/月/年/360–364日/不正/空）、設定の永続化、一括スターの成功・部分失敗・空選択・stale選択無視を XCTest で検証（全 96 件 green、iPhone 16 シミュレータ）。

## トレードオフと留保事項

- **月は 30 日固定・年は 365 日固定**の近似（web ADR-030 と同一）。概数表示として許容。
- アクセシビリティ（VoiceOver/Dynamic Type）は **issue #49 iOS で別途**対応。
