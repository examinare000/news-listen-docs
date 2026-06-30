# ADR-045: 連続再生・再生キュー（プレイリスト）

**ステータス:** 採用済み
**決定日:** 2026-06-30
**対象読者:** iOS / Web 開発者
**関連:** [ADR-022](022-server-side-playback-position-and-preferences.md)（再生位置同期）・[ADR-039](039-ios-background-playback-now-playing.md)（iOS バックグラウンド再生）・[iOS 設計書 §PodcastView](../design/ios-design.md)・[Web 設計書 §音声再生](../design/web-design.md)・GitHub issue #81

## 背景

従来の再生は 1 エピソード単位の手動操作で、連続再生やキュー（プレイリスト）が無かった。通勤・家事中の
「複数本をハンズフリーで流し続けたい」用途に直結するため、再生終了時の自動次再生とキュー管理を導入する。
バックエンドにキュー用エンドポイントは無く（PRD のスコープ外）、**キューはクライアント状態**として実装する。

決定論点は (A) キュー状態の置き場とテスト容易性、(B) 再生終了の検知方法、(C) 一覧からの再生（▶）がキューをどう扱うか。

## 検討した選択肢

### (A) キュー状態

| 選択肢 | メリット | デメリット |
|---|---|---|
| プレイヤー（AVPlayer / Audio）に直接持たせる | 近接 | プレイヤー依存でユニットテスト困難。自動次再生・並べ替えの検証が重い |
| **プレイヤー非依存の純粋キュー型 + 薄い配線**（採用） | 自動次再生・空キュー停止・並べ替えを純粋ロジックとして単体テストできる。iOS/Web で同一モデル | 配線（終了イベント→advance→次再生）を別途用意 |

### (B) 再生終了の検知

| プラットフォーム | 仕組み |
|---|---|
| iOS | `AVPlayerItem.didPlayToEndTimeNotification` を `play()` で当該 item に購読（`stopPlayback`/`deinit` で解除）。従来は終了検知が無かった |
| Web | `useAudioPlayer` の `'ended'` ハンドラに `onEnded` コールバックを追加。Provider が受けて advance |

### (C) 一覧 ▶ のキュー扱い

| 選択肢 | メリット | デメリット |
|---|---|---|
| キューを丸ごと置換（start） | 単純 | 利用者が組んだ待機列が ▶ タップで消える（データ損失的） |
| **現在の次に挿入して jump**（採用・レビュー指摘 C1/C2 を反映） | 待機列を保持しつつ即再生 | わずかに複雑 |

## 決定

- **(A)** プレイヤー非依存の純粋キュー型を両プラットフォームに実装する（iOS `PlaybackQueue.swift` / Web `lib/playbackQueue.ts`）。
  `items` と `currentIndex` を持ち、`advance`（次へ・末尾で停止）・`add`・`playNext`（現在の次に挿入）・`jump`・
  `remove`（現在を追従調整）・`reorderUpNext`（待機列の並べ替え）を純粋関数/メソッドで提供する。
- **(B)** 再生終了で `advance()` し、次があれば再生、無ければ停止する。iOS は `didPlayToEndTimeNotification`、
  Web は `useAudioPlayer` の `onEnded` で配線する。終端ではミニプレイヤーを閉じる。
- **(C)** 一覧/詳細の ▶ は、対象がキューにあればそこへ jump、無ければ**現在の次に挿入して jump** し、既存の待機列を保持する。
- 一覧から「次に再生」「キューに追加」、プレイヤーから「次へ」・キューの確認/並べ替え/削除を提供する。

## 理由

- 純粋キュー型により AC の「自動遷移・空キュー停止・並べ替え」を AVPlayer/Audio 非依存で確実に単体テストできる。
  iOS/Web で同一の状態モデル・規約を共有し、挙動差を防ぐ。
- 終了検知はプラットフォーム標準イベントに素直に乗せ、既存の Now Playing（ADR-039）・位置同期（ADR-022）と整合させる。
- ▶ でキューを破壊しない（挿入+jump）ことで、利用者が組んだプレイリストを尊重する。

## 結果（影響・既知の限界）

- **iOS:** `Podcast/PlaybackQueue.swift`（新規）・`Podcast/QueueSheet.swift`（キュー確認/並べ替え/削除）・
  `Podcast/PodcastViewModel.swift`（queue・終了監視・playNow/addToQueue/playNext/skip・自動次再生）・
  `Podcast/PodcastView.swift`（行タップ→playNow・コンテキストメニュー・キューシート）。PlaybackQueue/VM の単体テスト。
- **Web:** `lib/playbackQueue.ts`（新規）・`contexts/AudioPlayerContext.tsx`（キュー所有・自動次再生配線）・
  `hooks/useAudioPlayer.ts`（`onEnded`）・`components/AudioPlayerBar.tsx`（次へ・キューパネル）・
  `components/PodcastCard.tsx`（次に再生/キューに追加の導線）。キュー純粋ロジック + Provider 自動次再生の結合テスト。
- **既知の限界:** 詳細ページにはキュー導線を置かない（AC は一覧起点。フォローアップ）。Web の「ここから連続再生」
  一括投入（`setQueue`）は UI 未配線（将来）。アプリ強制終了時の保留はない（再生確定は即時）。
