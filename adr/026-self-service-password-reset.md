# ADR-026: パスワードリセットのセルフサービス（メール・短命使い捨てトークン）

**ステータス:** 採用済み
**決定日:** 2026-06-25
**対象読者:** バックエンド開発者 / 運用担当者
**関連:** [ADR-013](013-session-auth-and-user-management.md)（セッション認証）・[ADR-017](017-password-strength-and-session-rotation.md)（パスワード強度・セッション失効）・[ADR-019](019-csrf-double-submit-cookie.md)（CSRF）・[ADR-020](020-push-notification-web-push.md)（Notifier 抽象の範）・[ADR-015](015-audit-logging.md)（監査）・`backend/api/routers/auth.py`・`backend/shared/password_reset.py`・`backend/shared/email_sender.py`・issue #45（PRD §11）

## 背景

これまでパスワード変更は「認証済みユーザーの自己変更（現 PW 必須）」と「admin による直接リセット」のみで、**パスワードを忘れた利用者が自力で再設定する手段が無かった**（PRD §11 残課題）。メールでリセットリンクを送る一般的なフローを、既存スタック（Firestore + セッション認証）に追加する。トークン漏洩・ユーザー列挙・二重利用といったセキュリティリスクへの対策が要件。

## 検討した選択肢

| 選択肢 | メリット | デメリット |
|---|---|---|
| **短命・使い捨てトークン（ハッシュ保存）+ メール送信抽象（NoOp 降格）**（採用） | 既存スタックのみ。漏洩・列挙・二重利用を構造的に防ぐ。プロバイダ未設定でもテスト/ローカル可 | メール配送の最終的な到達性は外部依存 |
| admin 手動リセットのみ継続 | 追加実装ゼロ | セルフサービス不可（要件未達） |
| 外部 IdP（OAuth）へ委譲 | リセットを委譲 | 認証基盤の全面置換。スコープ過大 |

## 決定

### トークン（`PasswordResetToken`・`passwordResetTokens/{token_hash}`）
- 生トークンは `secrets.token_urlsafe(32)`。**DB には SHA-256 ハッシュのみ保存**（doc-id=ハッシュ。セッショントークンと同手法。生は保存しない）。
- **TTL は短命**（env `PASSWORD_RESET_TOKEN_TTL_MINUTES`・既定 30 分）。
- **使い捨て**: `consume_reset_token` が Firestore トランザクションで「未期限かつ `used_at is None`」を検証し `used_at=now` を原子更新する。二重消費はトランザクションで排除。

### エンドポイント（認証不要）
- `POST /auth/password/forgot`: **ユーザー列挙対策として常に 200・同一レスポンス**を返す。実在かつ `User.email` 有のときのみトークン発行→保存→メール送信。IP 単位のレート制限（env `PASSWORD_RESET_RATELIMIT_*`・既定 0=無効）。
- `POST /auth/password/reset`: トークンハッシュ照合→`consume_reset_token`（期限/使用済み検証）→`validate_password_strength`（[ADR-017](017-password-strength-and-session-rotation.md)）→`password_hash` 更新→**全セッション失効**（`delete_sessions_for_user`・強制ログアウト）。不正/期限/既用は**汎用 400**（原因秘匿）、弱 PW は 422。

### メール送信抽象（`EmailSender`・[ADR-020](020-push-notification-web-push.md) の Notifier に倣う）
- `EmailSender` Protocol + `SmtpEmailSender`（SMTP env 完備時）+ `NoOpEmailSender`（env 未設定時の既定・ログのみで **reset_url を出さない**＝生トークン漏洩防止）。`build_email_sender(env)` で自動降格。送信失敗は warning のみで**非致命**（要求受理は 200）。

### CSRF 免除（[ADR-019](019-csrf-double-submit-cookie.md)）
- `/auth/password/forgot`・`/auth/password/reset` は**未認証**（セッション Cookie 無し）の事前認証エンドポイントで、リセットトークン自体が CSRF 相当の保護を担う。`/auth/login` と同様に CSRF 既定免除に加える（double-submit cookie を要求するとログアウト中の利用者がリセット不能になるため）。

### データモデル・監査
- `User.email: str | None`（Optional・後方互換）。リセット送信先。
- `AuditAction` に `password_reset_requested` / `password_reset_completed` を追加。`details` に**平文 PW・生トークン・ハッシュを入れない**（[ADR-015](015-audit-logging.md)）。

## 理由

- ハッシュ保存・短命・使い捨て・列挙対策・全セッション失効を組み合わせ、リセットフロー固有の攻撃面（トークン漏洩・推測・二重利用・列挙・乗っ取り後の継続セッション）を多層で塞ぐ。
- メール送信を Protocol + NoOp 降格にすることで、プロバイダ未設定でもフロー全体をテストでき、外部依存をエッジに隔離する（VAPID/Notifier と同じ設計）。
- 純粋関数 `verify_reset_token(record, now)` とトランザクション `consume_reset_token` を分離し、時刻注入で決定的にテストできる。

## 結果（影響）

- 新規 `shared/password_reset.py`（純粋検証）・`shared/email_sender.py`（送信抽象）。`FirestoreClient` に `save_reset_token`/`get_reset_token`/`consume_reset_token`。`api/dependencies.py` に `get_email_sender`。
- env 追加: `PASSWORD_RESET_TOKEN_TTL_MINUTES`・`PASSWORD_RESET_RATELIMIT_MAX_REQUESTS`/`_WINDOW_SECONDS`・`PASSWORD_RESET_URL_BASE`・`SMTP_HOST`/`PORT`/`USERNAME`/`PASSWORD`/`FROM`（`.env.example`）。
- 受け入れ条件: 列挙対策（不在も 200）・トークンのハッシュ保存（生非保存）・期限切れ/既用拒否・弱 PW 422・リセット成功で PW 更新＋全セッション失効・メール非致命をテストで検証（`tests/test_password_reset.py`）。
- 実プロバイダ（SMTP/SendGrid 等）の接続は**デプロイ設定**（env/Secret Manager）。未設定時は NoOp で送信されない。

## トレードオフと留保事項

- **配送の到達性**: メール到達は外部依存。NoOp 時はリンクが送られない（ローカル/テスト用）。
- **列挙のタイミング差**: found 時のみトークン発行・送信で僅かに遅い。定数時間化は本 ADR の対象外（応答本文・ステータスは同一）。
- **メール経路の盗聴**: HTTPS 前提。短命トークンで窓を最小化するが、メール配送経路の保護は運用に委ねる。
- **email 逆引き未提供**: forgot は username 受け（`get_user_by_email` は作らない・YAGNI）。email ログインが必要になれば別 issue。
