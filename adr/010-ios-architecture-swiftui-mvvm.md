# ADR-010: iOS アーキテクチャに SwiftUI + MVVM + Swift Concurrency を採用する

**ステータス:** 採用済み（2026-06-16 更新: 状態管理を `@Observable` → `ObservableObject` に変更）
**決定日:** 2026-06-14
**最終更新日:** 2026-06-16
**対象読者:** iOS 開発者
**関連:** [iOS設計書 §1・§5（MVVM）](../design/ios-design.html)

## 背景

`ios/` 配下に新規ネイティブアプリを実装する。Web パリティと iOS 特有挙動（バックグラウンド再生等）を満たしつつ、`agent-rules/11-testing-strategy.md`（TDD・ViewModel 単体テスト）に適合する構成が必要。

## 検討した選択肢

| 選択肢 | メリット | デメリット |
|---|---|---|
| UIKit + MVC | 実績豊富 | 宣言的でなく記述量が多い。新規最小アプリには過剰。最小 iOS を下げる動機もない |
| SwiftUI + 素の View（状態を View に内包） | 着手が速い | ロジックが View に癒着しテスト困難。TDD と相性が悪い |
| **SwiftUI + MVVM + Swift Concurrency + `ObservableObject`** | View と状態ロジックを分離しテスト容易。標準のみで完結。`@StateObject`/`@EnvironmentObject` の枯れた配線 | MVVM の責務分割を規律として守る必要 |
| TCA 等の外部アーキテクチャ FW | 状態管理が厳密 | 追加依存・学習コスト。本規模には過剰 |

## 決定

SwiftUI（iOS 17+）を UI に、MVVM を基本アーキテクチャに採用する。非同期は Swift Concurrency（async/await）、状態は `ObservableObject` + `@Published`（`@StateObject`/`@EnvironmentObject` で配線）、ネットワークは標準 `URLSession`、音声は AVFoundation。外部依存は導入しない。`AppState` をグローバル共有状態とし、各画面に `ViewModel` を置く。並行性はプロジェクト既定を `nonisolated`（`SWIFT_DEFAULT_ACTOR_ISOLATION`）とし、`AppState`/`APIClient`/各 ViewModel に `@MainActor` を明示する。

## 理由

宣言的 UI で記述量を抑えつつ、ViewModel 分離により TDD（モック `URLSession` 注入）を成立させられる。標準フレームワークのみで依存管理コストを最小化できるため。

## 結果（影響）

- 最小対応 iOS は 17（`ContentUnavailableView` 等のモダン API 利用のため）。
- テストは具象 `APIClient` に `URLSessionProtocol`（`MockURLSession`）を注入して行う（専用の `APIClientProtocol` は設けない）。`APIClient` が `@MainActor` のためテストクラスも `@MainActor` 化する。
- View にビジネスロジックを書かない規律を維持する（`agent-rules/13-readability.md` 準拠）。
- 将来 iPad/macOS 展開時も SwiftUI 資産を流用できる。

## 更新（2026-06-16）

実装着手（Task 1〜4）に伴い、状態管理を当初の `@Observable`（Observation framework）から **`ObservableObject` + `@Published`** に変更した。

- **理由:** superpowers 実装プラン（`docs/superpowers/plans/2026-05-31-ios-app.md`）が `ObservableObject`/`@StateObject`/`@EnvironmentObject` ベースで設計されており、これに準拠した。`@Observable` への移行は将来の検討事項とする。
- **並行性の補足:** Xcode 16 テンプレート既定の `SWIFT_DEFAULT_ACTOR_ISOLATION = MainActor` だと `Codable` モデルの適合が main-actor 分離になり非分離テストからデコードできない。これを避けるため既定を `nonisolated` に変更し、UI/通信境界（`AppState`/`APIClient`/各 ViewModel）にのみ `@MainActor` を明示する方針とした。
- **テスト注入:** 当初想定の `APIClientProtocol` は導入せず、`URLSessionProtocol`（`MockURLSession`）の注入で代替する。
- ADR-008/009 等、他 ADR への影響はない。
</content>
