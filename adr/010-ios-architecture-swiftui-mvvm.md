# ADR-010: iOS アーキテクチャに SwiftUI + MVVM + Swift Concurrency を採用する

**ステータス:** 採用済み
**決定日:** 2026-06-14
**対象読者:** iOS 開発者
**関連:** [設計 §1・§3](../design/2026-06-14-ios-app-design.md)

## 背景

`ios/` 配下に新規ネイティブアプリを実装する。Web パリティと iOS 特有挙動（バックグラウンド再生等）を満たしつつ、`agent-rules/11-testing-strategy.md`（TDD・ViewModel 単体テスト）に適合する構成が必要。

## 検討した選択肢

| 選択肢 | メリット | デメリット |
|---|---|---|
| UIKit + MVC | 実績豊富 | 宣言的でなく記述量が多い。新規最小アプリには過剰。最小 iOS を下げる動機もない |
| SwiftUI + 素の View（状態を View に内包） | 着手が速い | ロジックが View に癒着しテスト困難。TDD と相性が悪い |
| **SwiftUI + MVVM + Swift Concurrency + @Observable** | View と状態ロジックを分離しテスト容易。標準のみで完結 | MVVM の責務分割を規律として守る必要 |
| TCA 等の外部アーキテクチャ FW | 状態管理が厳密 | 追加依存・学習コスト。本規模には過剰 |

## 決定

SwiftUI（iOS 17+）を UI に、MVVM を基本アーキテクチャに採用する。非同期は Swift Concurrency（async/await）、状態は `@Observable`（Observation）、ネットワークは標準 `URLSession`、音声は AVFoundation。外部依存は導入しない。`AppState` をグローバル共有状態とし、各画面に `ViewModel` を置く。

## 理由

宣言的 UI で記述量を抑えつつ、ViewModel 分離により TDD（モック `URLSession` / `APIClientProtocol` 注入）を成立させられる。標準フレームワークのみで依存管理コストを最小化できるため。

## 結果（影響）

- 最小対応 iOS は 17（`@Observable` / `ContentUnavailableView` 利用のため）。
- テストは `URLSessionProtocol` と `APIClientProtocol` の注入を前提に設計する。
- View にビジネスロジックを書かない規律を維持する（`agent-rules/13-readability.md` 準拠）。
- 将来 iPad/macOS 展開時も SwiftUI 資産を流用できる。
</content>
