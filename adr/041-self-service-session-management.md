# ADR-041: 自分のデバイス／セッション管理（一覧・個別失効）

**ステータス:** 採用済み
**決定日:** 2026-06-30
**対象読者:** バックエンド / Web / iOS 開発者
**関連:** [ADR-013](013-session-auth-and-user-management.md)（セッション認証の基盤）・[ADR-014](014-login-rate-limiting.md)（IP ハッシュ化方針）・[ADR-015](015-audit-logging.md)（監査ログ）・[ADR-035](035-passkey-webauthn-adoption.md)（credential の list/delete 流儀）・[backend 設計書 §6](../design/backend-design.md#6-認証ユーザー管理adr-013)・GitHub issue #84

## 背景

セッションは Firestore `sessions` に SHA-256 ハッシュで保持され（ADR-013）、admin による
ユーザー単位の一括失効はある（`delete_sessions_for_user`）。だが**利用者本人が自分の有効
セッション（ログイン中デバイス）を一覧し、特定の 1 つを失効させる手段が無い**。端末紛失や
不審ログイン時の自衛ができず、Passkey（ADR-035）と並ぶアカウント保護の基本機能が欠けていた。

決定すべき論点は (A) セッションを識別子としてどう公開するか、(B) 「最終利用時刻」をどう記録するか、
(C) IP・デバイス情報の保持方針、の 3 点。

## 検討した選択肢

### (A) セッション識別子

| 選択肢 | メリット | デメリット |
|---|---|---|
| **`session_id`（= SHA-256(token)）をそのまま識別子に使う**（採用） | 追加採番なし。SHA-256 は不可逆で、認証には生トークンが必要なため公開しても認証突破に使えない | ハッシュ値が API に露出する（ただし失効は所有権検証必須・preimage 困難） |
| 別途 opaque な公開 ID を採番 | 内部 doc-id を隠せる | doc-id 設計・ローテーション処理への波及が大きく、1〜5 人規模には過剰 |

### (B) 最終利用時刻（last_used_at）の更新

| 選択肢 | メリット | デメリット |
|---|---|---|
| 全認証リクエストで無条件更新 | 常に最新 | 高頻度な認証経路に毎回書き込みが乗り、コスト SLO（PRD §6・月次 5USD 以下）を圧迫 |
| **スロットル更新（既定 5 分）**（採用） | 書き込み回数を上限化しつつ実用十分な粒度。`get_session` で取得済みの値を使い追加リードなし | 最大 5 分の鮮度遅れ（許容） |

### (C) IP・デバイス情報

| 選択肢 | メリット | デメリット |
|---|---|---|
| 生 IP を保存 | 厳密な照合・地理情報化が可能 | PII を平文保持。漏洩時のリスク |
| **`hash_token(ip)` を保持し、クライアントには返さない**（採用・ADR-014 踏襲） | 生値を残さない。既存方針と一貫 | IPv4 は無塩ハッシュでも復元しうる（強い匿名化ではない）→「生値非保存」目的に限定 |
| デバイス名は User-Agent から best-effort 導出 | 追加依存なしで「Chrome on macOS」等を表示 | desktop-mode の iPad 等は誤判別あり（許容） |

## 決定

- **(A)** `session_id`（トークンの SHA-256 ハッシュ）をレスポンスの `id` として公開し、失効はこの id を指定する。
  失効は**所有権検証（`user_id` 一致）必須**で、他人・不在は `404`（存在秘匿）。Passkey の `delete_credential` と同流儀。
- **(B)** `get_current_user` が認証成功時に `last_used_at` を `SESSION_LASTUSED_THROTTLE_SECONDS`（既定 300 秒）で
  スロットル更新する。**ベストエフォート**（datetime 演算・書き込みの失敗で認証を妨げない）。
- **(C)** `Session` に `device_label` / `ip_hash` / `last_used_at` を追加（既存ドキュメント互換のため Optional+default）。
  `ip_hash` は生値非保存目的に限定しクライアントへ返さない。`device_label` は User-Agent から「ブラウザ on OS」を導出。
- エンドポイント（いずれも本人認証必須）:
  - `GET /auth/sessions` — 有効セッション一覧（期限切れ除外・最終利用降順）。現在のセッションはリクエスト由来トークンから算出し `current=true`。
  - `DELETE /auth/sessions/{session_id}` — 個別失効（所有権検証）。失効後そのセッションは次回 `401`。
  - `POST /auth/sessions/revoke-others` — 現在以外を一括失効（「他の全デバイスからログアウト」）。保持する現在セッションはリクエスト由来で算出。
- 失効は既存 `session_revoke` 監査アクションで記録し、`details.scope`（`self_single` / `self_others`）で admin 失効と区別する。

## 理由

- 1〜5 人規模で必要なのは「自分のログイン中デバイスを見て、怪しいものを切る」ことであり、不可逆な `session_id` の
  公開＋所有権検証つき失効で過不足なく満たせる。別 ID 採番の複雑さは便益に見合わない。
- 認証経路は最も高頻度な書き込み圧力源になりうる。スロットルで書き込みを上限化し、コスト SLO を守りつつ
  実用上十分な「最終利用」粒度を得る。取得済みセッションを使うため追加リードは発生しない。
- IP ハッシュ化は ADR-014 の既存方針と一貫。無塩ハッシュの限界を正直に「生値非保存」目的へ限定し、過大な
  匿名化保証を主張しない（クライアント非公開）。

## 結果（影響）

- **backend:** `Session` モデル・`issue_session`・`get_current_user` を拡張し、`FirestoreClient` に
  `list_sessions_for_user` / `revoke_session` / `delete_sessions_for_user_except` / `update_session_last_used` を追加。
  `.env.example` に `SESSION_LASTUSED_THROTTLE_SECONDS` を追記。一覧・失効・認可・UA 導出・スロットルの自動テストを追加。
- **web / iOS:** 設定画面に「ログイン中のデバイス」一覧と失効 UI（現在のセッションを明示）を追加する（別 PR）。
- **互換性:** 既存セッションドキュメントは新フィールドを持たないが Optional+default のため読み出し・一覧に支障なし。
