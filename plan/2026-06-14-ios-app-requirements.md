# iOS アプリ実装要件（plan / 要件）

**対象読者:** iOS 開発担当・バックエンド担当・レビュアー
**ステータス:** 実装中（Phase 1・Living Doc）— Task 1〜4 完了、Feed 以降は未着手
**最終更新日:** 2026-06-16
**関連:** [iOS設計書](../design/ios-design.html) / ADR [007](../adr/007-ios-direct-backend-access.md)・[008](../adr/008-ios-local-state-persistence.md)・[009](../adr/009-ios-signed-url-refetch.md)・[010](../adr/010-ios-architecture-swiftui-mvvm.md)・[011](../adr/011-podcast-generation-status.md) / [Web フロントエンド設計書](../design/web-design.md)

---

## 1. 概要（目的と背景）

`ios/` 配下に SwiftUI ネイティブアプリを実装する。現時点の Web UI（Next.js）と**機能同等（feature parity）**であることを第一目標とし、その上で iOS 特有の挙動（バックグラウンド再生・ロック画面操作・スワイプ操作・SafariView 等）に対応する。

バックエンド API は Web と共通（FastAPI / Cloud Run）。バックエンドは `X-API-Key` 認証 + デプロイ単位の `USER_ID` による単一ユーザー前提であり、iOS もこの契約に従う（マルチユーザー認証は本スコープ外）。

### 1.1 既存設計資産との関係（重要）

`docs/design/ios-design.html` に先行する iOS 設計案が存在するが、**実バックエンドとの契約乖離**がある。本要件・設計はこれを正とし、以下の乖離を是正する。

| ios-design.html の記述 | 実バックエンド | 本要件での扱い |
|---|---|---|
| `PATCH /podcasts/:id/position` で再生位置をサーバー保存 | **存在しない**。Web は localStorage に保存 | 端末ローカル（UserDefaults）に保存し Web と同等（[ADR-008](../adr/008-ios-local-state-persistence.md)） |
| `GET/POST/DELETE /subscriptions/:id`（ID 指定） | `GET/POST/DELETE /settings/sources`（**URL キー**・ID なし） | `/settings/sources` を使用。削除は `?url=` クエリ |
| `PUT /settings` で既定難易度・速度をサーバー保存 | **存在しない**。Web は localStorage 保存 | 端末ローカルに保存（[ADR-008](../adr/008-ios-local-state-persistence.md)） |
| `status` を見て生成中ポーリング | API レスポンスに `status` **未含有** | 生成状態表示はバックエンド変更が前提。Phase 1 は対象外（[ADR-011](../adr/011-podcast-generation-status.md)） |

---

## 2. スコープ

### 2.1 Phase 1（feature parity・本要件の主対象）
Web UI が現状提供する機能をすべて iOS ネイティブで実現する。

### 2.2 Phase 2（iOS 拡張・別ブランチ）
バックグラウンド再生の高度化、APNs 通知、オフライン再生、Keychain 移行、生成状態表示。Phase 2 は [PRD](../prd/2026-05-31-news-listen.md) の Phase 2 候補に追補する。

### 2.3 スコープ外
- マルチユーザー認証・サインイン基盤
- バックエンドの大規模な契約変更（`status` 追加等は [ADR-011](../adr/011-podcast-generation-status.md) で別途判断）
- iPad 最適化レイアウト（縦持ち iPhone を主対象。iPad はユニバーサルで動作すればよい）

---

## 3. 機能要件（Web パリティ・マトリクス）

各機能は「Web の挙動」を基準とし、iOS での実現方法と差分を明記する。受け入れ基準は §5 を参照。

### 3.1 初回セットアップ
- アプリ未設定時、API ベース URL と API キーの入力画面を表示する。
- 接続テスト（`GET /health` 相当）を提供する。
- 入力値は端末に永続化し、次回以降はセットアップをスキップする。
- Web の SetupModal に相当。Web のローカル Docker 事情（`http://api:8080`）は不要で、実機からは到達可能なホストを入力する。

### 3.2 Feed タブ
- `GET /feed` で記事一覧（最大 50 件、スコア降順）を取得・表示する。
- 各記事: ソース名・公開日時・タイトル・関連スコア（0〜1）バーを表示。
- Star 操作 → `POST /articles/{id}/star`、Dismiss 操作 → `POST /articles/{id}/dismiss`。
- 操作後は当該記事をリストから除去し、フィードバック（トースト相当）を出す。
- 記事タップで元記事 URL を開く。開き方は Settings の「記事の開き方」設定に従う（既定=アプリ内 `SFSafariViewController` / 外部 Safari）。
- 引っ張って更新（pull-to-refresh）、初回ローディング表示、エラー表示、空状態表示。

### 3.3 Podcast タブ
- `GET /podcasts` でエピソード一覧（最大 50 件、`created_at` 降順）を取得・表示する。
- 各エピソード: 難易度バッジ・種別（digest/single）・日本語イントロ抜粋・再生時間・作成日・続き再生位置を表示。
- タップで再生開始。再生中エピソードのインジケータ表示。
- 再生は共有プレイヤー（アプリ全体で 1 インスタンス）で行う。
- 詳細表示: 完全な日本語イントロ・難易度・所要時間・対象記事 ID 群。

### 3.4 音声プレイヤー
- 再生 / 一時停止、シーク（スライダー）、相対シーク（−15 秒 / +30 秒）。
- 再生速度（0.5 / 0.75 / 1.0 / 1.25 / 1.5 / 1.75 / 2.0 / 2.5 の 8 段階）。
- 音量調整と音量の永続化。
- 再生位置を**端末ローカルに**定期保存（Web は 10 秒間隔）。次回同エピソードで続きから再生。
- 署名付き URL の失効（1 時間）に対応するため、**再生直前にエピソードを再取得**してから読み込む（[ADR-009](../adr/009-ios-signed-url-refetch.md)）。
- エラー時はメッセージ表示し再生状態を復帰。

### 3.5 Subscriptions（RSS ソース管理）
> **実装方針:** 独立タブではなく **Settings タブ内のセクションとして実現**する（3 タブ構成 = Feed / Podcast / Settings）。設計書 §2・§3 と整合。本節の機能要件・受け入れ基準（AC-10/AC-11）は配置先が Settings になるだけで内容は不変。
- `GET /settings/sources` で一覧取得。`POST /settings/sources`（`{name, url}`）で追加。`DELETE /settings/sources?url=...` で削除。
- ソースは **URL をキー**として識別する（ID は存在しない）。
- 追加フォーム: 名前・URL の入力、クライアント側 URL 形式バリデーション、推奨ソースのクイック入力。
- エラー表示: 409（重複）「この URL は登録済みです」、422（不正フィード）「RSS として読み込めません」。
- 削除はスワイプ + 確認。

### 3.6 Settings タブ
- 既定の難易度（6 段階）と既定の再生速度（8 段階）を設定。**端末ローカルに保存**（[ADR-008](../adr/008-ios-local-state-persistence.md)）。
- **記事の開き方**（アプリ内 Safari / 外部 Safari）を選択。**端末ローカルに保存**。既定=アプリ内 Safari。Feed §3.2 のタップ動作に反映。
- RSS ソース管理（§3.5）。
- API エンドポイント URL・API キーの編集と接続テスト。
- ダーク/ライトテーマ切替（システム設定追従を既定）。

### 3.7 難易度レベル（バックエンドと一致必須）
`toeic_600` / `toeic_900`（既定） / `ielts_55` / `ielts_7` / `eiken_2` / `eiken_p1`。表示は 3 段階のトーン（易/中/難）バッジにマッピングする。

---

## 4. 非機能要件 / iOS 特有要件

### 4.1 iOS 特有挙動
- **バックグラウンド再生**: `AVAudioSession`（`.playback` / `.spokenAudio`）+ Background Modes（audio）。アプリが背面・画面ロックでも再生継続。
- **ロック画面 / コントロールセンター操作**: `MPNowPlayingInfoCenter` で曲情報、`MPRemoteCommandCenter` で再生/一時停止/スキップ。
- **スワイプ操作**: Feed は `swipeActions`（leading=Star、trailing=Dismiss）。Subscriptions は `onDelete`。
- **外部リンク**: 記事は `SFSafariViewController` で開く（外部 Safari に飛ばさない）。
- **割り込み対応**: 電話・他アプリ音声による中断/再開（`AVAudioSession` interruption 通知）。
- **空状態 / 未接続**: `ContentUnavailableView`（iOS 17）で統一。

### 4.2 セキュリティ
- API キーはログ・エラーに出力しない。
- Phase 1 は `UserDefaults`（@AppStorage）に保存し、Phase 2 で Keychain へ移行（[ADR-008](../adr/008-ios-local-state-persistence.md)、`agent-rules/12-security-guidelines.md` 準拠）。
- 入力 URL のスキーム検証（http/https のみ）。

### 4.3 品質・テスト
- `agent-rules/11-testing-strategy.md` / TDD 準拠。ViewModel・APIClient はモック `URLSession`（プロトコル注入）で単体テスト。
- テスト対象: API デコード/エラー処理、Star/Dismiss 後のリスト更新、再生速度の `AVPlayer.rate` 反映、再生位置のローカル保存/復元、RSS 追加・409 重複・削除。

### 4.4 対応環境
- 最小 iOS 17、Swift 5.10+、Xcode 16+。iPhone 縦持ち主対象。

---

## 5. 受け入れ基準（条件・実行・結果）

| # | 条件 | 実行 | 期待結果 |
|---|---|---|---|
| AC-1 | 未設定状態 | アプリ起動 | セットアップ画面が表示される |
| AC-2 | 有効な URL・キーを入力 | 接続テスト | 成功表示後、メイン画面（4 タブ）へ遷移 |
| AC-3 | Feed 表示中 | 記事を leading スワイプ | Star され `POST /articles/{id}/star` が送信、リストから除去 |
| AC-4 | Feed 表示中 | 記事を trailing スワイプ | Dismiss され `POST /articles/{id}/dismiss` が送信、リストから除去 |
| AC-5 | Feed 表示中 | 記事タップ | Settings の「記事の開き方」に従い、アプリ内 Safari（既定・`SFSafariViewController`）または外部 Safari で元記事が開く |
| AC-6 | Podcast 一覧 | エピソードをタップ | 再生直前にエピソードを再取得し、保存済み位置から再生開始 |
| AC-7 | 再生中 | アプリを背面化 / 画面ロック | 再生が継続し、ロック画面に曲情報と操作ボタンが出る |
| AC-8 | 再生中 | 速度を 1.5 に変更 | 即時に再生速度へ反映され、既定速度として永続化 |
| AC-9 | 再生を途中で停止し再度開く | 同エピソードをタップ | 前回位置の続きから再生される |
| AC-10 | Subscriptions | 既存 URL を追加 | 409 を受け「登録済み」エラーをインライン表示、シートは閉じない |
| AC-11 | Subscriptions | ソースをスワイプ削除 | 確認後 `DELETE /settings/sources?url=...` が送信、一覧更新 |
| AC-12 | Settings | 既定難易度を変更 | 端末に保存され、再起動後も保持 |
| AC-13 | 通信失敗時 | 任意のフェッチ | エラーが表示され、リトライ可能（クラッシュしない） |
| AC-14 | テスト | `xcodebuild test` 相当 | 全ユニットテスト PASS |

---

## 6. タスク分解（実装着手時に随時更新）

> Living Doc。完了したら `docs/plan/README.md` の方針に従い恒久ドキュメントへ反映し、本ファイルは削除する。
> タスク単位の手順・コミット履歴は [superpowers プラン](../superpowers/plans/2026-05-31-ios-app.md)（Task 1〜8）で追跡する。本節は要件視点の進捗サマリ。

- [x] `ios/NewsListenApp/` に Xcode プロジェクト（SwiftUI App）と最小ビルド導入
  - 補足: バックグラウンド再生 capability（`UIBackgroundModes=audio`）は未設定。音声再生実装（下記 AudioPlayerState）に合わせて追加予定。
- [x] Models（Codable）: Article / Podcast / RssSource — バックエンド契約に一致（snake_case → CodingKeys）
- [x] APIClient（`URLSessionProtocol` 注入）+ APIError + エンドポイント定義 + テスト
- [x] AppState + 初回セットアップ + ローカル永続化（UserDefaults）
- [ ] Feed（View / ViewModel / Row / スワイプ / SafariView）+ テスト　← 現在プレースホルダのみ
- [ ] Podcast 一覧・詳細（View / ViewModel / Row）+ テスト　← 現在プレースホルダのみ
- [ ] AudioPlayerState（AVPlayer・バックグラウンド・位置ローカル保存・再取得）+ テスト
- [ ] Now Playing / Remote Command（ロック画面操作）
- [ ] Subscriptions（一覧・追加・削除・バリデーション・エラー）+ テスト
- [ ] Settings（難易度・速度・API 設定・テーマ）+ テスト　← 現在プレースホルダのみ
- [ ] 仕上げ: 空状態・エラー・トースト相当・ダークモード確認

### 6.1 実装メモ・設計書との差分（要レビュー）

実装着手後に生じた、iOS設計書（`docs/design/ios-design.html`）との差分を記録する。状態管理・タブ構成・テスト注入・Deployment Target は本対応で**設計書/ADR-010 を実装に合わせて更新済み**。音声再生のみ Task 6 未着手のため設計書 §6 を据え置き、再調整事項として残す。

| 項目 | 設計書の旧記述 | 実装 | 対応 |
|---|---|---|---|
| 状態管理 | `@Observable`（Observation framework） | `ObservableObject` + `@Published` + `@StateObject`/`@EnvironmentObject` | superpowers プラン準拠。**設計 §1・§3 / ADR-010 を更新済み**。 |
| タブ構成 | 4 タブ（Feed / Podcast / **Subscriptions** / Settings） | 3 タブ（Feed / Podcast / Settings、RSS 管理は Settings に内包） | **設計 §2・§3 を 3 タブに更新済み**。§3.5 も注記済み。 |
| アクター分離 | 明記なし（ViewModel は `@MainActor`） | プロジェクト既定を `SWIFT_DEFAULT_ACTOR_ISOLATION = nonisolated` とし、`AppState`/`APIClient`/各 ViewModel に個別 `@MainActor` を付与 | Xcode テンプレート既定の MainActor 分離だと Codable モデルが非分離テストから使えないため。**設計 §1 / ADR-010 に追記済み**。 |
| テスト注入 | `APIClientProtocol` を差し替え | 具象 `APIClient` に `URLSessionProtocol`（`MockURLSession`）を注入 | **設計 §5 / ADR-010 を更新済み**。ViewModel テストも同方式。 |
| Deployment Target | iOS 17+ | アプリ/テスト/UIテスト全構成を iOS 17.6 に統一 | 生成時にテスト系のみ 26.4 だった不整合を是正。 |
| 音声再生の責務 | `AudioPlayerState`（独立 `@Observable`）を `AppState` 経由で全体共有 | **未実装（Task 6 予定）**。プランでは `PodcastViewModel` に再生機能を内包する構成 | **要再調整**。単一プレイヤー共有の要件（§3.4 / 設計 §6）と整合するか Task 6 着手前に判断（設計 §6 は据え置き）。 |
</content>
