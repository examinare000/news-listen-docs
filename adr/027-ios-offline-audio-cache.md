# ADR-027: iOS オフライン再生（音声ローカルキャッシュ）

**ステータス:** 採用済み
**決定日:** 2026-06-25
**対象読者:** iOS 開発者
**関連:** [ADR-009](009-ios-signed-url-refetch.md)（署名付き URL の再取得）・[ADR-010](010-ios-architecture-swiftui-mvvm.md)（SwiftUI + MVVM）・[ADR-008](008-ios-local-state-persistence.md)（端末ローカル保存）・`ios/NewsListenApp/NewsListenApp/Networking/AudioCacheManager.swift`・issue #46（PRD §5.2・§11 P1）

## 背景

iOS は生成済み Podcast の音声を、毎回サーバの**署名付き URL（1 時間失効）を AVPlayer で直接ストリーム再生**していた。ローカルキャッシュが無く、**オフラインでは生成済みエピソードも再生できなかった**（PRD §5.2/§11 P1）。生成済み音声を端末にキャッシュしてオフライン再生を可能にする。web の PWA 対応は本 issue の対象外（iOS 優先）。

## 検討した選択肢

| 選択肢 | メリット | デメリット |
|---|---|---|
| **手動ダウンロード→FileManager キャッシュ→ローカル URL を AVPlayer に渡す**（採用） | オフライン再生可。I/O・到達性を抽象化してテスト可能。署名 URL 失効と独立 | ダウンロード操作 UI が要る |
| `URLCache`（HTTP キャッシュ）任せ | 実装最小 | 署名 URL は失効するとキャッシュも無効・オフライン保証なし |
| ストリーミング DL（部分保存） | 大容量に強い | 実装複雑。1 エピソード数 MB 想定では過剰（YAGNI） |

## 決定

- **`AudioCacheManager`**（新規 Service・状態を持たない）が音声 blob のローカルキャッシュを管理する。保存先 `Caches/NewsListenApp/audio-cache/`、ファイル名 `{podcast.id}.mp3`。`cachedURL/isCached/cache/remove/cacheSize` を提供。
  - **path traversal 防御**: `podcast.id` をファイル名にする前に `[A-Za-z0-9_-]` のみ許可し、不正 id は `AudioCacheError.invalidId` を throw（`../` 等を Caches に書かせない）。
- **`NetworkMonitor`**（新規・`NWPathMonitor` 駆動）が `isOnline` を提供。`NetworkMonitoring` プロトコルで抽象化しテストでスタブ注入。
- **`APIClient.downloadAudio(from:)`**: 外部署名 URL（GCS 等）から音声データを取得する。**X-API-Key・Authorization を一切付けず baseURL も連結しない**（API ゲート用の秘密を外部ストレージに送らない）。署名 URL の再取得は `fetchPodcast(id:)`（[ADR-009](009-ios-signed-url-refetch.md)）。
- **再生 URL の決定は純粋関数** `resolvePlaybackURL(for:isOnline:cacheManager:)`: キャッシュ有→ローカル `file://` URL／キャッシュ無＋オンライン→署名 URL／キャッシュ無＋オフライン→`nil`（再生不可・ダウンロードを促す）。AVPlayer は副作用が大きいためこの決定ロジックのみをテスト対象にする（[ADR-010](010-ios-architecture-swiftui-mvvm.md) の MVVM・テスト容易性）。
- **`PodcastViewModel`**: `download(podcast:)`（署名 URL 再取得→ダウンロード→キャッシュ保存、二重起動防止）・`downloadState(for:)`・`syncDownloadedState()`・`removeDownload(podcast:)`。`downloadedIds`/`downloadingIds` は `@Published private(set)`。
- I/O・到達性・ネットワークは3抽象（`FileManagerProtocol`／`NetworkMonitoring`／既存 `URLSessionProtocol`）でのみ外部に触れ、テストは実 FS・実ネットワーク・実 AVPlayer に触れない。

## 理由

- 署名 URL 失効に依存しない**自前のローカルキャッシュ**を持つことで、オフライン再生を保証できる。URLCache では署名失効後に再生不能になる。
- 再生 URL の決定を純粋関数に切り出し、I/O を3抽象に隔離することで、AVPlayer を起動せずにオフライン分岐を網羅テストできる。
- `downloadAudio` がヘッダを付けないことで、API ゲート用の `X-API-Key` を外部ストレージへ漏らさない。path traversal 防御で Caches 外への書き込みを構造的に防ぐ。

## 結果（影響）

- 新規 `AudioCacheManager`・`FileManagerProtocol`・`NetworkMonitoring`/`NetworkMonitor`。`APIClient` に `fetchPodcast(id:)`/`downloadAudio(from:)`、`APIEndpoint` に `podcast(id:)`。`PodcastViewModel` にダウンロード・キャッシュ・オフライン分岐。UI（`PodcastRowView` にダウンロードボタン＝未DL/DL中/DL済）。
- 受け入れ条件: 「オフラインで生成済みエピソードが再生可能」。キャッシュ保存/有無/削除/サイズ・path traversal throw・`resolvePlaybackURL` の3分岐・download フロー・downloadAudio が API キーを付けない、を XCTest で検証（実機シミュレータで全緑）。

## トレードオフと留保事項

- **容量管理は別 issue**: 自動退避・容量上限は本タスク非対象（`cacheSize()`/`remove()` の seam のみ用意）。バックエンドのストレージ管理は [ADR-025](025-storage-management-safe-cleanup.md)。
- **Data 経由 DL**: 既存 `URLSessionProtocol`/`MockURLSession` を流用するため `downloadTask`（ファイル）でなく `data`（メモリ）で取得。1 エピソード数 MB 想定で許容。将来大容量化すれば拡張。
- **再生位置はサーバ保存（[ADR-022](022-server-side-playback-position-and-preferences.md)）のまま**。オフライン中の位置同期は対象外。
- **web PWA は未対応**: 本 ADR は iOS のみ。web のオフライン対応は別 issue。
- **設計書（ios-design）への反映**: `ios-design.html` は Markdown 移行（`docs/plan/2026-06-24-md-canonical-migration.md` の Phase 2b）待ちのため本 ADR を正本とし、HTML は手編集しない。
