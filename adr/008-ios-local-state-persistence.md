# ADR-008: 再生位置・既定設定・認証情報を端末ローカルに保存する

**ステータス:** 採用済み
**決定日:** 2026-06-14
**対象読者:** iOS 開発者
**関連:** [要件 §1.1](../plan/2026-06-14-ios-app-requirements.md)・`agent-rules/12-security-guidelines.md`

## 背景

先行設計案（`docs/design/ios-design.html`）は再生位置を `PATCH /podcasts/:id/position`、既定難易度・速度を `PUT /settings` でサーバー保存する想定だったが、**これらのエンドポイントはバックエンドに存在しない**。現行 Web は再生位置・既定速度・テーマ・API 設定をすべてブラウザ `localStorage` に保存しており、サーバーには永続化していない。`UserPrefs`（Firestore）に `default_difficulty` 等のフィールドは存在するが、これを読み書きする API は公開されていない。

## 検討した選択肢

| 選択肢 | メリット | デメリット |
|---|---|---|
| バックエンドに設定・位置の API を新設しサーバー保存 | 複数端末で同期できる | バックエンド変更が必要で「1 ブランチ = 1 目的」に反する。Web との parity を超える先行投資。スコープ拡大 |
| **端末ローカル（UserDefaults）に保存（Web と同等）** | バックエンド変更不要・Web と挙動一致・実装最小 | 端末間同期は不可（Web も同様の制約） |

## 決定

再生位置（キー `podcast_position:{id}`）、既定難易度、既定再生速度、音量、API ベース URL、API キーを端末ローカル（`UserDefaults` / `@AppStorage`）に保存する。サーバー側設定 API は新設しない。

API キーは Phase 1 では `UserDefaults` に保存し、**Phase 2 で Keychain へ移行**する（`UserDefaults` は暗号化されないため）。

## 理由

Web と機能同等にすることが Phase 1 の目的であり、Web 自体がローカル保存方式のため。サーバー設定 API の新設はスコープを超え、別個の意思決定（バックエンド変更）を要するため見送る。

## 結果（影響）

- 端末間で再生位置・設定は同期されない（Web と同じ制約。要件として許容）。
- 再生位置のキー規約は Web（`podcast_position:{id}`）と一致させる。
- 認証情報の Keychain 移行は [PRD](../prd/2026-05-31-news-listen.md) の Phase 2 候補に追補する。
- サーバー保存が将来必要になった場合は、バックエンド API 追加とあわせ新 ADR で上書きする。
</content>
