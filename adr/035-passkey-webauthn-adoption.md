# ADR-035: Passkey（WebAuthn/FIDO2）認証の採用

**ステータス:** 採用済み
**決定日:** 2026-06-26
**対象読者:** バックエンド / Web / iOS 開発者
**関連:** [PRD §6](../prd/2026-05-31-news-listen.md)・[backend 設計書（認証）](../design/backend-design.md)・[ADR-013](013-session-auth-and-user-management.md)（セッション認証）・[ADR-016](016-cors-and-security-headers.md)（CSRF）・[ADR-017](017-password-strength-and-session-rotation.md)（パスワード強度）・[ADR-019](019-csrf-double-submit-cookie.md)（CSRF Double-submit）・[ADR-026](026-self-service-password-reset.md)（パスワードリセット）

## 背景

ユーザー認証は現在、username + password のセッション方式（ADR-013）で実装されている。パスワード認証の課題は：

1. **フィッシング耐性の欠如**: パスワードを詐欺的な偽サイトに入力すればハッシュ値にかかわらず盗まれる。
2. **リスト型攻撃への脆弱性**: 他サービスでの流出データを試す攻撃に対し、ユーザーの「使い回し」習慣が効く。
3. **利便性と強度のトレードオフ**: 人間が覚えやすいパスワードほど辞書攻撃に弱い。

一方、**Passkey（WebAuthn/FIDO2）** は以下の強みを持つ：

- **フィッシング耐性**: 認証器が RP ID（サイトの識別子）を厳格検証し、詐欺サイトに応答しない（仕様レベルの防御）。
- **リスト型攻撃耐性**: デバイスに秘密鍵が保有され、サービス側で盗まれるシークレットがない（鍵・サーバー署名で検証）。
- **クローン検知**: `sign_count` の後退で改ざんされた認証器を検知し、その credential を拒否する。

**新規認証手段として Passkey を追加** し、既存のパスワード認証と併存させる（フォールバック・段階的採用）。ユーザーは任意で Passkey を登録でき、複数デバイスに credential を持つことで冗長性を確保できる。

## 検討した選択肢

### (A) Passkey 採用の是非

| 選択肢 | メリット | デメリット |
|--------|---------|----------|
| Passkey 非採用（パスワード継続） | 実装が最小。ユーザーの習慣変更不要 | フィッシング・リスト型攻撃への対抗力が弱い。セキュリティ向上の道が限定的 |
| **Passkey 追加採用（パスワード併存）** | フィッシング・リスト型・クローン攻撃に強い。ユーザーが段階的に採用可能。最低1認証手段（パスワード）でロックアウト防止 | backend/web/iOS で WebAuthn 実装が必要。複数の credential 管理が加わる |
| Passkey 一本化（パスワード廃止） | 最高のセキュリティ。管理シンプル | 既存ユーザーの切り替え强制・フォールバック経路なし。アカウント回復が困難。本プロジェクト規模（1〜5人）には過剰 |

### (B) 認証器からの credential 保存先・検証方式

| 選択肢 | メリット | デメリット |
|----------|---------|----------|
| credential を Firestore コレクション保存、credential_id で直引き（O(1)） | シンプルで高速。公開鍵は API 応答で返さない設計で秘密性損なわず | credential_id がユーザーに可視・予測される可能性。列挙防止が必要 |
| credential を認証器内のみ保有、毎回ユーザーに表示させ選択 | サーバーに credential 痕跡がない | credential 数が増えると UX が悪化。失効・削除の管理が複雑 |
| **`SHA-256(credential_id)` を doc-id として保存** | O(1) 直引き。credential_id の平文を保存しない（hash により列挙困難） | hash 用の digest アルゴリズム依存。衝突リスクは SHA-256 では無視可能 |

### (C) challenge（登録・認証時の一時トークン）の管理

| 選択肢 | メリット | デメリット |
|----------|---------|----------|
| challenge を長期保存（セッション継続まで） | 実装が簡単。多重タブ対応が楽 | リプレイ攻撃のリスク。challenge が漏洩しても長期有効 |
| **challenge をワンタイム利用（検証時に get+delete）** | リプレイ攻撃の中核的防御。challenge 漏洩後も即座に無効化 | Firestore トランザクション必須。多重タブ・並行リクエストで delete の先着が失敗 → ユーザー側では無視（login optionsやregister optionsを新規取得） |

### (D) session の発行・transport

| 選択肢 | メリット | デメリット |
|----------|---------|----------|
| **既存セッション発行機構に合流** | パスワード・Passkey 認証の後流（session 管理・CSRF・監査）を共通化。`issue_session` ヘルパに抽出し重複排除 | 既存コード修正が必要 |
| Passkey 専用トークン（JWT・stateless） | 既存コード への変更最小 | session・CSRF・監査の分断。将来的に footgun（矛盾する権限・セッション状態） |

### (E) CSRF 対策（login と register/delete）

| 選択肢 | メリット | デメリット |
|----------|---------|----------|
| 全エンドポイントで CSRF 必須 | 最も安全 | `POST /auth/passkey/login/options` が CSRF を要求すると、未認証ユーザーが CSRF トークンを事前に取得しなければならず、UX が複雑 |
| **login系（options/verify）は CSRF 免除、register/delete は CSRF 必須** | login は未認証・ステートレスで進行。register（スコープ変更）・delete（credential 削除）は credential 管理として CSRF で保護 | 設計の明確な理由付けが必要（既存 ADR-019 で `CSRF_EXEMPT_PATHS` に `/auth/password/forgot` / `/auth/password/reset` 先例あり。register/delete も同じロジック・既存パスワード変更は `PATCH /admin/users/{username}` で CSRF 必須） |

### (F) sign_count 後退検知時の挙動

| 選択肢 | メリット | デメリット |
|----------|---------|----------|
| credential 削除（キツい） | クローン確実性が高い。ユーザーに危機感が伝わる | ユーザーが意図せず credential を失う可能性（複数デバイス・バックアップ回復）。ログイン失敗で credential を失うのは報復的 |
| **認証拒否（ログイン失敗）** | 問題を認識させ、credential を保持。credential 再確認・削除は ユーザーの自意の選択肢に委ねる | クローンされていてもユーザーが削除まで進まない可能性。但し、以降のログイン試行で毎回拒否されるため気づく |

## 決定

- **(A) Passkey を追加認証手段として導入する。** パスワード認証と並存させ、ユーザーが任意で Passkey を登録・利用できる。フィッシング耐性・リスト型耐性の向上が目的（[§ 背景](#背景)）。将来的に「最後の認証手段削除は拒否」ポリシーを追加し、passwordless 化への道を開く（本 ADR の範囲外）。

- **(B) Firestore `credentials` コレクションに公開鍵 + メタデータを保存、`doc_id = SHA-256(credential_id)` で O(1) 直引き。** `credential_id` を平文で保存しない（hash 値のみ保存）ことで credential 列挙を困難にする。API レスポンスでは public_key を返さない（検証は server-side のみ）。

- **(C) challenge をワンタイム方式で運用。** Firestore `webauthnChallenges` に短命（有効期限 `WEBAUTHN_CHALLENGE_TTL_MINUTES`、既定 10 分）で保存し、検証時に `get + delete` をトランザクションで実行（リプレイ防止の中核）。多重タブでの並行 login/register options 呼び出しは許容（challenge 失効時はクライアント側で新規取得）。

- **(D) 認証成功後は既存のセッション発行に合流。** `issue_session` ヘルパに認証方式に依らないセッション発行機構を抽出し、パスワード・Passkey の両者が同じ session トークン + Cookie / Bearer 輸送を使う。token + httpOnly Cookie / Bearer dual transport。TTL は既存 `SESSION_TTL_HOURS`。下流の CSRF・監査ログ・レート制限は既存ルール（ADR-019・ADR-015・ADR-018）に従う。

- **(E) login系（`/auth/passkey/login/options`・`/auth/passkey/login/verify`）は CSRF 免除。** 未認証ユーザーが options リクエストで challenge を取得し、ステートレスに login/verify を進める。register・delete・credentials list は CSRF 必須（スコープ・状態変更）。既存の `CSRF_EXEMPT_PATHS` 環境変数に `/auth/passkey/login/*` を追記（ADR-019）。

- **(F) sign_count 後退を検知したら認証拒否（`401`）。** 以降のログイン試行でも拒否され、ユーザーは気づく。credential 削除は UI から自発的に選択させる。以降のログイン成功で `sign_count` を更新し、正常な credential に回復可能。

- **RP 設定を環境変数化** （`WEBAUTHN_RP_ID`・`WEBAUTHN_RP_NAME`・`WEBAUTHN_ORIGIN`（CSV）・`WEBAUTHN_TIMEOUT_MS`）。RP ID = `news-listen.example.com`（ユーザーが登録時に設定）。origin 厳格検証で CSRF/詐欺サイト対策。

- **最低1認証手段の保持要件はなし** （`User.password_hash` が必須のままだが、Passkey 全削除でもロックアウトしない）。passwordless 化（「最後の認証手段削除拒否」）は将来の大型更新。

- **usernameless フロー（ユーザー列挙対策）**: `/auth/passkey/login/options` は `username` を送らない discoverable credential 専用フロー。登録済み credential の情報をクライアント側で保有（同期・キャッシュ）し、サーバーの `allowCredentials` は常に空配列（credential_id 漏洩防止・ユーザー列挙防止）。

- **監査ログ記録** (`passkey_register`・`passkey_used`・`passkey_removed`)。秘密情報（public_key・credential_id）は details に入れない。

- **使用ライブラリ**:
  - backend: `py_webauthn`（登録・検証のコア・RP 設定・challenge 処理）
  - web: `@simplewebauthn/browser`（クライアント実装）
  - iOS: `AuthenticationServices`（os 標準・Passkey サポート）

## 理由

- **フィッシング耐性**: Passkey は仕様レベルで RP ID を厳格検証し、詐欺サイトへの応答を拒否する。パスワード + HTTPS では達成できない防御（HTTPS と Passkey は直交した防御）。

- **リスト型攻撃耐性**: 秘密鍵がサーバーに保存されず、デバイスに閉じ込められる。流出データとの照合では何も得られない。

- **クローン検知**: `sign_count` 後退は改ざん・クローン・時間旅行の検知器。検知後の「認証拒否」はユーザーが気づくまでの段階的対応（credential 削除は ユーザーの自発的選択に委ねる）として、手厚い。

- **パスワード併存**:本プロジェクト規模（1〜5人）でユーザーへの一括切り替え強制は不適切。Passkey の追加オプション化で段階的採用を可能にし、フォールバック経路も確保。

- **ワンタイム challenge**: RFC 8174 および Chromium Reference Security Standard で "Challenge must be used only once"（§5.1.3）と明記。複数リクエストで再利用された challenge は無視すべき。Firestore トランザクションで get+delete を原子的に実行し、同一 challenge 再利用攻撃・リプレイを防ぐ。

- **challenge.type 多層検証**: verify 時に challenge の `type`（registration / authentication）をクライアント `clientDataJSON` から抽出し、 Firestore 保存時の `operation` フィールドと照合。不一致は登録リクエスト時 400、ログインリクエスト時 401 で拒否。challenge タイプ詐称（例: 登録済み credential で registration を試行）を防ぐ。

- **challenge 期限判定**: Pydantic によるレスポンス再構築後、サーバー側 `datetime.utcnow()` と challenge の `expires_at`（タイムスタンプ）を比較。型安全な datetime 比較で期限超過判定。

- **セッション合流**: session・CSRF・監査ログの既存機構を再利用。Passkey 専用トークンを避けることで、権限・監査の一貫性を保つ。

- **SHA-256(credential_id) 格納**: 平文の credential_id を保存しない。DB 漏洩時に credential_id リストから credentialattestationobject を再構成される危険を低減。実装は `hashlib.sha256(credential_id).hexdigest()`。

- **CSRF 免除パス**: login は未認証・ステートレスで進行（options → verify）。challenge 自体が一意の csrf トークン的役割を果たす。register・delete（スコープ変更）は CSRF で保護。既存 ADR-026（パスワードリセット）の `CSRF_EXEMPT_PATHS` 先例に従う。

## 結果（影響）

### バックエンド

**スキーマ・リソース追加:**

- Firestore `credentials` コレクション
  - doc-id: `SHA-256(credential_id)`
  - フィールド: `credential_id`（平文）、`public_key`（PEM形式）、`sign_count`（整数）、`transports`（配列・"usb" 等）、`registered_at`（タイムスタンプ）、`last_used_at`（タイムスタンプ or null）、`user_id`（外部キー）、`friendly_name`（ユーザー指定名）
  - インデックス: `(user_id, registered_at DESC)`（credentials list 用）

- Firestore `webauthnChallenges` コレクション（短命）
  - doc-id: SHA-256 ハッシュ（ランダム部）
  - フィールド: `challenge`（base64url エンコード）、`user_id`（or null、login ではロックなし）、`operation`（"register" / "login"）、`created_at`（タイムスタンプ）、`expires_at`（TTL）
  - インデックス: 不要（get+delete only）

**API エンドポイント追加:**

| Method | Path | 説明 | CSRF | 認証 |
|--------|------|------|------|------|
| POST | `/auth/passkey/register/options` | 登録の challenge 取得 | 不要 | 不要 |
| POST | `/auth/passkey/register/verify` | 登録の検証・credential 保存 | 不要 | 不要 |
| POST | `/auth/passkey/login/options` | ログインの challenge 取得 | 不要 | 不要 |
| POST | `/auth/passkey/login/verify` | ログインの検証・セッション発行 | 不要 | 不要 |
| GET | `/auth/passkey/credentials` | 登録済み credential 一覧 | 不要 | 必須 |
| DELETE | `/auth/passkey/credentials/{credential_id}` | credential 削除 | 必須 | 必須 |

**リクエスト/レスポンス形式（概要）:**

- `POST /auth/passkey/register/options`:
  - Request: `{ "username": "user123", "display_name": "Alice iPhone" }`
  - Response: `{ "challenge": "base64url(...)", "rp": { "id": "news-listen.example.com", "name": "News Listen" }, ... }`
- `POST /auth/passkey/register/verify`:
  - Request: `{ "challenge": "...", "attestationObject": "...", "clientDataJSON": "..." }`（clent からのレスポンス）
  - Response: `{ "status": "success", "credential_id": "...", "friendly_name": "Alice iPhone" }`
- `POST /auth/passkey/login/options`:
  - Request: `{ "username": "user123" }` または `{}` （usernameless）
  - Response: `{ "challenge": "...", "rp": { "id": "..." }, "timeout": 60000, "allowCredentials": [...] }`（登録済み credential リスト）
- `POST /auth/passkey/login/verify`:
  - Request: `{ "challenge": "...", "id": "...", "rawId": "...", "response": { "authenticatorData": "...", "clientDataJSON": "...", "signature": "..." } }`
  - Response: `{ "status": "success", "session_token": "...", "user_id": "...", ... }`（session 発行・Cookie SET）
- `GET /auth/passkey/credentials`:
  - Response: `[ { "credential_id": "...", "friendly_name": "Alice iPhone", "registered_at": "2026-06-26T...", "last_used_at": "..." }, ... ]`
  - 公開鍵・sign_count を返さない

**セキュリティ実装:**

- **challenge ワンタイム検証**: `webauthnChallenges` から get+delete を Firestore トランザクションで実行。存在しない・期限超過は 400 / 401。
- **challenge.type 多層検証**: verify リクエストの `clientDataJSON` をデコードし `type` フィールドを抽出。Firestore 保存時の `operation`（registration / authentication）と照合。不一致は登録時 `400`、ログイン時 `401` で拒否。
- **challenge 期限判定**: Pydantic レスポンス再構築後、サーバー側 `datetime.utcnow()` と challenge `expires_at`（タイムスタンプ）を型安全に比較。
- **sign_count 後退検知**: Credential 検証時に `attestedCredentialData.sign_count >= saved_sign_count` を確認。後退を検知したら `401`（認証拒否・credential 削除なし）。
- **RP ID / origin 厳格検証**: `py_webauthn.verify_registration_response` / `verify_authentication_response` が自動実行。RP ID・origin が環境変数と一致しないなら例外。
- **public_key は API 返却しない**: credential 検証は server-side のみ。「credentials list」の API では public_key を含めない。
- **秘密情報非ログ**: 監査ログは `passkey_register` / `passkey_used` / `passkey_removed` として記録。details 内に public_key・credential_id・sign_count を入れない（ユーザー・timestamp・状態のみ）。

**環境変数（新規）:**

| 変数 | 既定 | 説明 |
|------|------|------|
| `WEBAUTHN_RP_ID` | `localhost` | Relying Party ID（FQDN。登録時のサイト識別子） |
| `WEBAUTHN_RP_NAME` | `News Listen` | RP Display Name（登録時のダイアログに表示） |
| `WEBAUTHN_ORIGIN` | `http://localhost:3000` | Origin（カンマ区切り複数対応。厳格検証） |
| `WEBAUTHN_TIMEOUT_MS` | 60000 | challenge タイムアウト（ミリ秒） |
| `WEBAUTHN_CHALLENGE_TTL_MINUTES` | 10 | challenge 有効期限（分） |

**実装モジュール:**

- `backend/shared/webauthn_client.py`: `py_webauthn` ラッパー。`verify_registration_response` / `verify_authentication_response` / `generate_challenge`。
- `backend/api/routers/auth_passkey.py`: Passkey エンドポイント（register/options/verify・login/options/verify・credentials・delete）。
- `backend/api/dependencies.py`: `get_current_user` へ Passkey session も対応させる（既存 session と統一）。
- `backend/shared/security.py`: Passkey 追加に伴うセッション発行を `issue_session` ヘルパに抽出（認証方式に依存しない）。

### Web フロントエンド

- `web/lib/webauthn.ts`: `@simplewebauthn/browser` ラッパー。`startRegistration` / `startAuthentication`。
- `web/components/AuthModal.tsx`: ログインフロー拡張。「パスワードログイン」「Passkey ログイン」タブ。
- `web/components/AccountSection.tsx`: Passkey 管理 UI（登録済み credential 一覧・削除・新規登録）。
- `web/lib/api.ts`: Passkey API 呼び出し統合。既存 `fetchApi` で対応。
- Web Push（[ADR-020](020-push-notification-web-push.md)）と統合：Passkey 登録完了時に通知（任意）。

### iOS アプリ

- `iOS/Models/AuthModels.swift`: Passkey request/response 構造体。
- `iOS/Services/AuthenticationService.swift`: `AuthenticationServices` 統合。`performAutoFillAssistedRequests` / `beginAutoFillAssistedRegistration`。
- `iOS/Views/LoginView.swift`: ログインフロー拡張。「パスワード」「Passkey」セグメント。
- `iOS/Views/AccountSettingsView.swift`: Passkey 管理 UI。
- `iOS/Services/SessionStore.swift`: Passkey session も Keychain に保存（既存 session と統一）。

### 既存コードの変更

- `backend/api/main.py`: `CSRF_EXEMPT_PATHS` に `/auth/passkey/login/*` を追記（ADR-019 環境変数）。
- `backend/api/dependencies.py`: `issue_session` ヘルパ抽出（パスワード・Passkey 統一）。
- `backend/shared/firestore_client.py`: `webauthnChallenges` / `credentials` コレクション操作メソッド追加。
- `backend/api/routers/auth.py`: `POST /auth/login` は既存のままパスワード認証に特化（Passkey 用は別ファイル）。
- `backend/shared/audit_logs.py` / `api/routers/audit.py`: Passkey 操作の監査ログ出力（`passkey_register` / `passkey_used` / `passkey_removed`）。

### テスト・マイグレーション

- 既存ユーザーに Passkey 登録を促すメッセージを設定画面に追記（バージョンアップ後）。
- パスワード認証は変わらず機能。credential を複数登録したユーザーは任意で Passkey ログインを選択可能。
- ローカル開発: `.env.local` で `WEBAUTHN_ORIGIN=http://localhost:3000`・`WEBAUTHN_RP_ID=localhost` に設定。

### 残課題・将来

- **Passwordless 化**: 「最後の認証手段削除拒否」ポリシー（user.password_hash を null 許可）は別タスク。
- **バックアップ回復**: Passkey を失ったユーザーの回復フロー（メールリセット・セキュリティキー等）は [ADR-026](026-self-service-password-reset.md) で対応。
- **監査・コンプライアンス**: Passkey 登録・使用・削除ログは [ADR-015](015-audit-logging.md) で統合管理（本 ADR は記録対象を定義のみ）。
- **多言語・国別対応**: discoverable credential の表現は認証器の実装に依存。RP_NAME が多言語対応する場合は環境変数化（将来検討）。
