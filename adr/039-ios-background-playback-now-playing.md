# ADR-039: iOS バックグラウンド再生 + ロック画面/コントロールセンター操作統合

**ステータス:** 採用済み
**決定日:** 2026-06-29
**対象読者:** iOS 開発者・バックエンド開発者
**関連:** [ADR-010](010-ios-architecture-swiftui-mvvm.md)（iOS アーキテクチャ）・[ADR-022](022-server-side-playback-position-and-preferences.md)（再生位置サーバー保存）・[issue #79](../../issues/79)・[ios#28](../../issues/28)・`ios/NewsListenApp/NewsListenApp/Podcast/PodcastViewModel.swift` ほか

## 背景

iOS ユーザーが Podcast を再生中に画面をロックしたり別アプリへ切り替えたりしても、音声が再生され続ける必要がある（バックグラウンド再生）。またロック画面やコントロールセンターの再生操作（再生/一時停止・スキップ・速度変更・シーク）が iOS の標準 UI で効くことが、iOS アプリの基本的な使いやすさの前提である。

これらの機能は AVPlayer 単体では提供されず、`AVAudioSession` の設定・`MPNowPlayingInfoCenter`（ロック画面表示）・`MPRemoteCommandCenter`（リモート操作）の連携が必須である。

## 検討した選択肢

| 選択肢 | メリット | デメリット |
|---|---|---|
| **バックグラウンド有効化 + Now Playing + リモートコマンド統合** | iOS ユーザーが標準的な使い方（ロック画面操作）で再生できる。リモートコマンド登録時に `[weak self]` クロージャを使い、シングルトン保持による memory leak を回避 | `MPRemoteCommandCenter` / `MPNowPlayingInfoCenter` の副作用が多い。割り込み・ルート変更の処理が必要 |
| バックグラウンド再生のみ（Now Playing・リモートコマンド無し） | 実装が小さい | ロック画面が真っ暗で操作不可。ユーザーが再生を止めたい場合、アプリを開く必要があり UX が劣悪 |

## 決定

iOS バックグラウンド再生を完全に実装し、以下の 4 要素を統合する:

### A. Info.plist の UIBackgroundModes 設定

```xml
<key>UIBackgroundModes</key>
<array>
    <string>audio</string>
</array>
```

背面・画面ロック・別アプリ切り替え後も音声再生を継続する。

### B. AVAudioSession 設定

`configureAudioSession()` で以下を実行:

```swift
try AVAudioSession.sharedInstance().setCategory(.playback, mode: .spokenAudio)
try AVAudioSession.sharedInstance().setActive(true)
```

- `.playback` カテゴリ: マナーモード（消音スイッチ）の影響を受けず再生される。
- `.spokenAudio` モード: スピーチに最適な音量・周波数設定。
- エラーは致命的でなく（音量が小さくなる程度）、再生自体は継続。

### C. MPNowPlayingInfoCenter での Now Playing 情報管理

2 段階のアプローチで効率と正確さを両立:

**離散イベント時（再生/一時停止・シーク・速度変更）:**
- `updateNowPlayingInfo()` で辞書全構築。タイトル・難易度・総時間・経過・再生速度・再生状態を全セット。
- `NowPlayingInfo.make()` 純粋関数に委譲し、ロック画面表示ロジックをテスト容易化。

**周期的な経過更新（0.5 秒ごと）:**
- `updateNowPlayingElapsed()` で既存辞書の経過/総時間のみ上書き。
- `addPeriodicTimeObserver` の high-frequency callback から呼び、辞書全構築のコスト（メモリ割り当て・キー dup）を避ける。

辞書のキー:
- `MPMediaItemPropertyTitle`: 日本語イントロテキスト（無ければ "ニュースポッドキャスト"）
- `MPMediaItemPropertyArtist`: 難易度（e.g., "初級"）
- `MPMediaItemPropertyPlaybackDuration`: 総時間（秒、有限・正のみ）
- `MPNowPlayingInfoPropertyElapsedPlaybackTime`: 経過時間（秒、0 以下を 0 にクリップ）
- `MPNowPlayingInfoPropertyPlaybackRate`: 再生速度（再生中は倍率、一時停止中は 0 で進行を止める）

### D. MPRemoteCommandCenter での リモートコマンド登録

ロック画面・コントロールセンター・Siri からの操作を受け取り、ViewModel メソッドへ配線。

**登録パターン（memory leak 回避）:**
```swift
let target = command.addTarget { [weak self] event in
    MainActor.assumeIsolated {
        guard let self else { return .commandFailed }
        // ViewController メソッド呼び出し
    }
}
remoteCommandTargets.append((command, target))
```

`[weak self]` クロージャでシングルトンへの強参照を避け、解除トークンを配列に保持して `deinit` で確実に削除する。

**登録コマンド:**

| コマンド | 操作 | 処理内容 |
|---|---|---|
| `playCommand` | "再生" | 未再生時は no-op（`noSuchContent`）、再生中なら no-op |
| `pauseCommand` | "一時停止" | 再生中時は no-op、未再生時は no-op |
| `togglePlayPauseCommand` | "再生/一時停止トグル" | 状態反転 |
| `skipBackwardCommand` | "戻る 15s" | `PlaybackConstants.skipBackwardSeconds` でシーク |
| `skipForwardCommand` | "進む 30s" | `PlaybackConstants.skipForwardSeconds` でシーク |
| `changePlaybackPositionCommand` | "スライダー操作" | 任意位置へシーク |
| `changePlaybackRateCommand` | "速度選択" | `PlaybackConstants.speeds` から選択 |

スキップ秒数・再生速度は `PlaybackConstants` で集約し、UI（AudioPlayerView）と リモートコマンド で一致させ、片側変更による齟齬を防ぐ。

### E. 割り込み（AVAudioSession.InterruptionNotification）・ルート変更対応

`InterruptionPolicy` ヘルパ enum で判定ロジックを純粋化し、テスト容易化。

**割り込み（電話着信等）:**
- `began`: 再生中フラグを `wasPlayingBeforeInterruption` に保存し、一時停止。
- `ended`: `shouldResume` オプション確認 + `wasPlayingBeforeInterruption` が true なら再開。セッション再有効化後に再生。

**ルート変更（イヤホン抜去等）:**
- `oldDeviceUnavailable`: 旧デバイス喪失（イヤホン抜去等）なら一時停止。
- 他の理由（BT 切換等）は何もしない。

### F. 再生位置のサーバー同期

[ADR-022](022-server-side-playback-position-and-preferences.md) で導入された `PATCH /podcasts/{id}/position` へ、`currentTime` を 15 秒ごとに同期。

- `startPlaybackPositionSync()` で Timer を開始。
- `syncPlaybackPositionIfNeeded()` で非同期に API 呼び出し。失敗時はサイレント（ネットワーク一時的不具合対応）。
- 再生停止時に `stopPlayback()` → `syncPlaybackPositionIfNeeded()` で最終同期。

## 理由

### メモリ安全性（リモートコマンド登録）

`MPRemoteCommandCenter` はシングルトンであり、`addTarget(self, action:)` 形式で登録すると `self` への強参照が残り、`deinit` が呼ばれず memory leak が生じる。`[weak self]` クロージャと解除トークン保持により、VM が解放される際に登録を自動削除できる。

### 効率（Now Playing 更新戦略）

ロック画面の経過表示は 0.5 秒ごとに更新される。毎回辞書全構築（メモリ割り当て・キー dup）すると CPU/電池を浪費するため、経過のみ上書きする軽量更新パターンを采用。タイトル・難易度等の変化は稀で、離散イベント時のみ全構築で十分。

### テスト容易性（InterruptionPolicy / NowPlayingInfo）

割り込み判定と Now Playing 辞書組み立てをシングルトンから切り離し、純粋関数 enum で実装。`AVAudioSession` / `MPNowPlayingInfoCenter` との依存を除き、ユニットテストで副作用なく検証可能。

### 定数共有（PlaybackConstants）

スキップ秒数・再生速度を 1 ヶ所に集約し、UI と リモートコマンド で一致させる。片側修正による齟齬回帰を防止。

## 結果（影響）

- `Info.plist`: `UIBackgroundModes` に `audio` を追加（既存・新規プロジェクト両対応）。
- `PodcastViewModel.swift`: 以下を新規/拡張
  - `configureAudioSession()`（AVAudioSession 設定）
  - `configureBackgroundPlayback()`（リモートコマンド・通知の初期化）
  - `configureRemoteCommands()`（MPRemoteCommandCenter 登録、[weak self] クロージャ方式）
  - `registerAudioNotifications()`（割り込み・ルート変更の購読）
  - `updateNowPlayingInfo()`（辞書全構築）
  - `updateNowPlayingElapsed()`（軽量更新）
  - `handleInterruption()` / `handleRouteChange()`（割り込み・ルート変更処理）
  - `startPlaybackPositionSync()` / `syncPlaybackPositionIfNeeded()`（再生位置定期同期）
  - `deinit`（リモートコマンド・通知の確実な解除）
  - プロパティ: `backgroundPlaybackConfigured` / `wasPlayingBeforeInterruption` / `remoteCommandTargets` / `audioNotificationObservers` / `syncTimer`
- `NowPlayingInfo.swift`: 新規（純粋ヘルパ enum）
  - `NowPlayingInfo.make()`（辞書組み立て）
  - `NowPlayingInfo.title()`（ロック画面タイトル決定）
  - `InterruptionPolicy.shouldResume()` / `shouldPause()`（割り込み判定）
- `PlaybackConstants.swift`: 新規（定数集約）
  - `speeds`（0.5 ～ 2.5 倍、8 段階）
  - `skipBackwardSeconds`（15 秒）
  - `skipForwardSeconds`（30 秒）
- テスト (`NowPlayingInfoTests.swift`): 純粋関数のユニットテスト
  - `make()` の出力を検証（タイトル・速度・経過・total 時間等）
  - `shouldResume()` / `shouldPause()` の判定を検証
- 対応する backend API: [ADR-022](022-server-side-playback-position-and-preferences.md) の `PATCH /podcasts/{id}/position` を既に採用。
- Web パリティ: Web は `audio` タグのネイティブコントロールで簡易的なバックグラウンド機能を備える（詳細は Web 側 ADR 参照）。

## トレードオフと留保事項

### Now Playing 表示の遅延

0.5 秒の周期更新なため、ロック画面の経過は実時間より最大 0.5 秒遅延する。高精度が必要な場合（e.g., スポーツ中継のリアルタイム同期）は `addPeriodicTimeObserver` の周期を短縮可能だが、電池消費増大とのトレードオフ。

### 再生位置同期の遅延

15 秒周期でサーバーへ同期するため、ユーザーが再生中に複数端末を切り替えた場合、最大 15 秒の位置ずれが生じうる。UX 上の問題は軽微（続き再生の誤差範囲）だが、必要に応じて周期短縮が可能。

### リモートコマンド登録成功の未検証

`command.addTarget` は戻り値の解除トークンを返すが、登録自体が常に成功するとは限らない。失敗時の例外処理は省略（iOS 標準 API のため滅多に失敗しない前提）。
