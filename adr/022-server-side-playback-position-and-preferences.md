# ADR-022: 再生位置・既定設定をサーバー保存し端末間で同期する

**ステータス:** 採用済み
**決定日:** 2026-06-25
**対象読者:** バックエンド / Web / iOS 開発者
**関連:** [ADR-008](008-ios-local-state-persistence.md)（本 ADR が一部上書き）・[ADR-013](013-session-auth-and-user-management.md)（user_id パーティション）・`backend/api/routers/podcasts.py`・`backend/api/routers/settings.py`・backend PR #21（issue #12）/ #13（issue #13）・web #12 / #13

## 背景（ADR-008 の留保の発火）

[ADR-008](008-ios-local-state-persistence.md) は再生位置・既定難易度・既定再生速度を端末ローカル（Web=localStorage、iOS=UserDefaults）に保存し、サーバー側 API は新設しないと決定した。理由は「Web と機能同等にすることが Phase 1 の目的であり、Web 自体がローカル保存方式のため」で、同 ADR は結果として **「サーバー保存が将来必要になった場合は、バックエンド API 追加とあわせ新 ADR で上書きする」** と留保していた。

その後、端末間での続き再生・設定共有が要件化（issue #12・#13）し、マルチユーザー認証（[ADR-013](013-session-auth-and-user-management.md)）で `user_id` パーティションが整ったため、サーバー保存の前提条件が揃った。本 ADR はこの留保を発火させ、再生位置・既定設定の保存方針を端末ローカルからサーバー保存へ上書きする。認証情報のローカル保存（Keychain 移行を含む）は本 ADR の対象外で、ADR-008 の決定を維持する。

## 検討した選択肢

| 選択肢 | メリット | デメリット |
|---|---|---|
| 端末ローカル保存を継続（ADR-008 現状維持） | バックエンド変更不要 | 端末間で再生位置・設定が同期されない。要件（続き再生）を満たせない |
| **サーバー保存 API を新設し `userPrefs`／`podcasts` に永続化** | 端末間で同期できる。難易度を生成パラメータの正本にできる | バックエンド変更が必要。クライアント申告値の検証が要る |

## 決定

### A. 再生位置のサーバー保存（issue #12）

`PATCH /podcasts/{id}/position` を新設し、再生位置を `Podcast.playback_position_seconds`（`podcasts` コレクション）に永続化する。

- 所有権チェック（`podcast.user_id != user_id` は 404）後に保存する。
- クライアント申告値は信用せず、`duration_seconds` に **clamp** する。
- レスポンスは更新後の `PodcastResponse`（署名付き URL 付き）。
- 端末ローカルのキー規約（`podcast_position:{id}`）はクライアントのオフライン／即時反映用キャッシュとして併存しうるが、正本はサーバー値とする。

### B. 既定設定のサーバー保存（issue #13）

`GET·PUT /settings/preferences` を新設し、既定難易度・既定再生速度・ダイジェスト設定を `userPrefs` コレクション（`user_id` ドキュメント）に永続化する。

- `PUT` は部分更新（`exclude_none=True` で指定フィールドのみ反映、他は保持）。
- `default_difficulty` は記事 Star 時の Podcast 生成パラメータの**正本**となる（生成側は env 未指定時に `prefs.default_difficulty` へフォールバック）。生成済み Podcast の `difficulty` は不変。

## 理由

1. **端末間同期**: ログイン中ユーザーは Web／iOS のどちらでも続きから再生でき、既定設定も共有される（ADR-008 が許容していた制約を要件側が解消）。
2. **生成パラメータの一貫性**: 難易度をサーバーの正本にすることで、Star 時に確保する processing 行（[ADR-021](021-podcast-generation-status-visualization.md)）と generator の難易度が一致し、重複行を防ぐ。
3. **前提条件の充足**: ADR-013 の `user_id` パーティションにより、ユーザー単位の安全な永続化が可能になった。

## 結果（影響）

- ADR-008 の「再生位置・既定設定はサーバー保存しない」決定を**上書き**する。ADR-008 はステータスを「一部廃止」に更新し、認証情報のローカル保存方針のみ有効として残す。
- API 契約に以下を追加（backend v0.3.0／v0.4.0）:

  | エンドポイント | 用途 |
  |---|---|
  | `PATCH /podcasts/{id}/position` | 再生位置を保存（`duration_seconds` に clamp、所有権 404） |
  | `GET /settings/preferences` | 既定難易度・再生速度・ダイジェスト設定を取得 |
  | `PUT /settings/preferences` | 上記を部分更新 |

- データモデル: `Podcast.playback_position_seconds`（廃止予定を撤回し正式採用）・`userPrefs.default_difficulty` ほかが API 経由で読み書きされる。
- Web は `lib/api.ts` の `updatePosition` / `getPreferences` / `updatePreferences` で BFF 経由同期する。
- iOS は再生位置・既定設定をサーバー値へ移行する余地が生まれる（ローカル保存からの移行スコープは別途）。

## 追記（2026-06-29）

本 ADR で「iOS はローカル保存からサーバー値への移行スコープは別途」とした留保について、**ios#27（commit `c50287a`、マージ済み）で実現**した。

- iOS アプリが `PATCH /podcasts/{id}/position` へ再生位置を定期同期（15 秒間隔）
- 既定難易度・再生速度をサーバー（`/settings/preferences`）から取得・反映
- ローカル UserDefaults からサーバー管理への移行が実装済み

iOS バックグラウンド再生（[ADR-039](039-ios-background-playback-now-playing.md)）でも同期タイマーが実装され、画面ロック中・別アプリ切り替え時も定期的にサーバーへ位置を通知する。
