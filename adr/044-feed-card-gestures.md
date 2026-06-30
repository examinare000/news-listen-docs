# ADR-044: フィードカードのジェスチャ操作と遅延コミット undo（iOS）

**ステータス:** 採用済み
**決定日:** 2026-06-30
**対象読者:** iOS 開発者
**関連:** [ADR-040](040-ios-editorial-design-system.md)（Editorial デザインシステム）・[ADR-033](033-ios-relative-time-and-bulk-star.md)（Feed の Star/一括操作）・[ADR-034](034-ios-accessibility-voiceover-dynamic-type.md)（アクセシビリティ）・[iOS 設計書 §FeedView](../design/ios-design.md)・GitHub issue #111

## 背景

フィードはアプリのメインループ（記事を選ぶ → Podcast 化して聴く）の入口。従来は SwiftUI `List` の
`.swipeActions`（leading=Star / trailing=Dismiss）だったが、片手・直感的なジェスチャ中心に刷新し、
Star→生成 / Dismiss の判断を高速化したい。AC（issue #111）:
右スワイプ→Star / 左スワイプ→Dismiss、タップ→全文展開、ダブルタップ→ソース表示、
スワイプの方向・閾値の可視化（指追従・色・アイコン・触覚）、ジェスチャ競合なし、誤操作の取り消し、
FeedViewModel のユニットテスト。

決定論点は (A) 高度なスワイプ表現をどう実装するか、(B) サーバに un-star/un-dismiss API が無い中での取り消し。

## 検討した選択肢

### (A) スワイプ実装

| 選択肢 | メリット | デメリット |
|---|---|---|
| `List.swipeActions` 継続 | 標準・VoiceOver 自動公開 | 指追従 offset・任意閾値・方向別アフォーダンス・触覚を出せない（AC 不達） |
| **`ScrollView`+`LazyVStack` の自作 `SwipeableArticleCard` + `DragGesture`**（採用） | 閾値・指追従・色/アイコン/触覚を完全制御。タップ/ダブルタップ/スワイプを 1 カードに統合 | List のスワイプ自動 VoiceOver 公開を失う → `accessibilityAction` で補う。縦スクロール競合に配慮要 |

### (B) 取り消し（undo）

| 選択肢 | メリット | デメリット |
|---|---|---|
| サーバに un-star/un-dismiss を追加 | 確定後も取り消せる | backend 改修（#111 の iOS スコープ外）。生成済み Podcast の扱いが複雑 |
| 即時コミット + ローカル再表示の擬似 undo | 単純 | サーバ状態は戻らず誤誘導 |
| **楽観削除 + 遅延コミット**（採用） | サーバ未送信なので真に取り消せる。生成も走らない | 確定タイミングの設計（タイマー/別操作/リフレッシュ/バックグラウンド）と整合が必要 |

## 決定

- **(A)** `List` を `ScrollView`+`LazyVStack` に置き換え、各記事を `SwipeableArticleCard` で描画する。
  `DragGesture` で右スワイプ→Star・左スワイプ→Dismiss（閾値 110pt 超で確定、未満はスプリングバック）。
  指追従 `offset`・方向別の背景色（Star=金 `DSColor.star` / Dismiss=赤 `DSColor.danger`）・アイコン・
  閾値到達で `sensoryFeedback`。タップ→全文展開（`lineLimit` 解除 + インライン Star/Dismiss ボタン）、
  ダブルタップ→`SafariView`。`onTapGesture(count:2)` を先に登録して単/ダブルを解決し、`DragGesture` は
  `simultaneousGesture` + 横優位ガードで縦スクロールと共存。VoiceOver には `accessibilityAction` で
  Star/Dismiss を公開する（List のスワイプ自動公開の代替）。
- **(B)** Star/Dismiss は **楽観削除 + 遅延コミット**。`FeedViewModel` が記事を即時に一覧から外して
  `pendingAction` に積み、サーバ送信（`commitPending`）は「取り消し猶予の経過（4 秒）・別の Star/Dismiss・
  `loadFeed`（リフレッシュ）・`scenePhase` のバックグラウンド遷移・画面離脱」のいずれかまで遅延する。
  確定前は「取り消す」トーストの `undoLast()` で元の位置へ戻す。`commitPending` は `await` 前に
  `pendingAction` を解除し再入を防ぐ（複数経路の同時到来でも 1 回のみ送信）。
- 連続スワイプのラグを避けるため、新操作は**先に楽観削除**してから直前操作を確定送信する。

## 理由

- AC の「方向・閾値の可視化（指追従・色・アイコン・触覚）」は `List.swipeActions` では実現できず、自作 `DragGesture`
  が必須。代償の VoiceOver 後退は `accessibilityAction` で補完する。
- サーバに un-action が無いため、真の取り消しは「まだ送っていない」遅延コミットでしか成立しない。確定を
  リフレッシュ/別操作/バックグラウンドにも紐付けることで、未送信のまま記事が再出現する不整合や取りこぼしを防ぐ。

## 結果（影響・既知の限界）

- **iOS:** `Feed/SwipeableArticleCard.swift`（新規）、`Feed/FeedView.swift`（`ScrollView` 化・undo トースト・
  `scenePhase` コミット）、`Feed/FeedViewModel.swift`（`star`/`dismiss` の遅延コミット化・`undoLast`・
  `commitPending`・`expandedId`）。FeedViewModel の楽観削除/取り消し/確定/連鎖確定/リフレッシュ整合/展開の
  ユニットテストを追加（XCTest）。
- **既知の限界:** アプリを 4 秒猶予内に強制終了すると未送信操作は失われる（楽観削除はローカルのみのため
  次回 `loadFeed` で記事は復活＝サーバ破壊はなく benign）。`ScrollView` 内 `DragGesture` の縦スクロール競合と
  ジェスチャ感は実機での微調整余地がある（`simultaneousGesture`+横優位ガードで緩和）。
