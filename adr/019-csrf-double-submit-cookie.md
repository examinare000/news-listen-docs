# ADR-019: Web Cookie 認証の CSRF 対策（double-submit cookie）

**ステータス:** 採用済み
**決定日:** 2026-06-25
**対象読者:** バックエンド / Web 開発者
**関連:** [ADR-016](016-cors-and-security-headers.md)（CORS・セキュリティヘッダ／本 ADR で「将来課題」の CSRF を解決）・[ADR-013](013-session-auth-and-user-management.md)（httpOnly Cookie 認証の防御を補完）・`backend/api/middleware/csrf.py`・`web/lib/api.ts`・backend PR #26 / web PR #9（issue #40）

## 背景

httpOnly Cookie でセッション認証を保護する（[ADR-013](013-session-auth-and-user-management.md)）一方で、**ブラウザが自動的に Cookie をリクエストに付与する仕様**を悪用した CSRF（Cross-Site Request Forgery）攻撃が残る。

悪意サイトから被害者を経由して本サイト API への状態変更リクエスト（POST / PUT / DELETE）を送信しても、ブラウザは被害者の Cookie を自動で付与するため、API は正規リクエストと区別できない。特に Web の BFF（後発フェーズで CORS を活用して撤去予定）と異なり、**モバイル SDK が直接バックエンド API へのアクセスをサポート**する設計では CSRF 脆弱性が顕在化する。

[ADR-016](016-cors-and-security-headers.md) では「CSRF 対策は将来課題」と留保していた。本 ADR でそれを **double-submit cookie** 方式で解決する。

## 検討した選択肢

| 選択肢 | メリット | デメリット |
|---|---|---|
| **Double-submit cookie（採用）** | ステートレス。トークンが Cookie と Header に両方あれば正規リクエスト。ブラウザのクロスオリジンリクエストでは Header が付かないため CSRF をブロック | X-CSRF-Token ヘッダが必須。サブドメイン共有 Cookie 時に脆弱性あり（本実装は `Secure/SameSite=Lax` で緩和） |
| CSRF トークン（サーバー保持） | 標準的。トークンを使い捨てやすい | ステートフル（Firestore にトークン保存必要）。スケール・キャッシュ無効の複雑性 |
| SameSite Cookie のみ | 設定1行。モダンブラウザで CSRF をブロック | 旧ブラウザに無効。iOS 等でセマンティクスが曖昧 |

## 決定

### A. Double-submit cookie トークン仕様

- **生成**: `secrets.token_urlsafe(32)` で URL-safe な Base64 エンコード文字列（43 文字程度）。暗号的に強い（256 bits）。
- **配置**: 非 httpOnly Cookie `csrf_token`（JavaScript から読み取り可能）と同時に、クライアントが `X-CSRF-Token` ヘッダで送り返す。両者が一致すれば正規リクエスト。
- **Cookie 属性**:
  - `HttpOnly: False`（JavaScript でアクセス可）
  - `SameSite: Lax`（クロスサイト Top-Level Navigation では付与。同一サイト POST / 外部サイトリンククリックからの GET では付与しない）
  - `Secure: {SESSION_COOKIE_SECURE 準拠}`（本番 HTTPS のみ）
  - `Max-Age: 604800`（7 日、セッション TTL と同期）
  - `Path: /`（全パス対象）

### B. ミドルウェア（`api/middleware/csrf.py`）

ASGI ミドルウェア `CsrfMiddleware`：

**免除条件**（以下いずれかなら CSRF チェックをスキップ）:
- **安全メソッド**: GET / HEAD / OPTIONS（副作用なし）
- **Bearer 認証**: `Authorization: Bearer ...` ヘッダ付き（iOS・サーバー間など。case-insensitive 判定）
- **免除パス**: `CSRF_EXEMPT_PATHS` で指定（既定 `/auth/login`）

**検証フロー**:
1. リクエストから `csrf_token` Cookie と `X-CSRF-Token` ヘッダを抽出
2. `hmac.compare_digest` で定時間比較（タイミング攻撃対策）
3. 不一致または欠落 → 403 Forbidden（エラーボディに詳細を含めない）
4. 一致 → ルーターへ委譲

**環境変数**:
- `CSRF_PROTECTION_ENABLED`（既定 `false`）: 本番で true に設定。ローカル開発・テストで無効化可能
- `CSRF_EXEMPT_PATHS`（既定 `/auth/login`）: カンマ区切り。未設定・空白のみなら安全側に `/auth/login` へ縮退（ログインフロー保証）

### C. ミドルウェア適用順序

**バックエンド（`backend/api/main.py`）:**

```python
# ミドルウェアスタック（下記の順で add_middleware で登録 → 逆順実行）
# 最内： CSRF → CORS → SecurityHeaders → 最外
```

**WHY**: CORS が OPTIONS プリフライト応答を生成するが、**CSRF は OPTIONS を免除する**ため先に CORS 処理。SecurityHeaders は最外に置いて 403 エラー応答にもセキュリティヘッダを付与（ブラウザが理由を読める）。

### D. Cookie 発行タイミング

- **ログイン成功時** (`POST /auth/login` レスポンス): `Set-Cookie: csrf_token=...`
- **補填機構** (`GET /auth/me`): トークン未発行の場合、レスポンスに新規トークンを発行（無停止移行対応）

### E. Web / BFF 側の統合（`web/lib/api.ts`）

- **状態変更メソッド** (POST / PUT / DELETE): `csrf_token` Cookie を読み取り、`X-CSRF-Token` ヘッダに付与して送信
- **BFF パススルー** (`web/app/api/backend/[...path]/route.ts`): リクエストの `X-CSRF-Token` ヘッダを保存して下流へ転送

### F. CORS 設定の追加

`CORS_ALLOWED_HEADERS` に `X-CSRF-Token` を追加（プリフライト OPTIONS で許可）。

## 理由

- **Double-submit cookie**: ステートレスで Firestore トランザクション不要。Web ブラウザが同一オリジンリクエストで Header を自動付与しないため、クロスオリジンからの CSRF 攻撃をブロック。
- **Bearer 免除**: iOS など API トークン認証を使う SDK にとって CSRF は当てはまらない（トークン詐称不可）。免除することで iOS の実装を簡潔化。
- **SameSite=Lax**: モダンブラウザでの CSRF 攻撃軽減。ただし旧ブラウザ・エッジケース対応のため double-submit と併用。
- **hmac.compare_digest**: タイミング攻撃で暗号値を推測される脆弱性を回避。定時間の比較で、一致/不一致の処理時間を統一。
- **ミドルウェア順序**: CORS 後に CSRF を置くことで、プリフライト（OPTIONS）が CSRF チェック内側に入らない。403 エラーにも CORS ヘッダが乗ることで、ブラウザが正しく拒否理由を解釈可能。

## 結果（影響）

### モジュール

- **新規:** `backend/api/middleware/csrf.py`（トークン生成・検証・ミドルウェア実装）
- **修正:** `backend/api/main.py`（CSRF ミドルウェア登録）、`backend/api/dependencies.py`（Cookie 処理あれば）
- **Web:** `web/lib/api.ts`（状態変更メソッドで Header 付与）、`web/app/api/backend/[...path]/route.ts`（BFF パススルー）

### API

- **ログインエンドポイント** (`POST /auth/login`):
  - リクエスト: `username` / `password`（既存）
  - レスポンス: `Set-Cookie: csrf_token=...` を追加（既存 `session_token` と並行）
  
- **認証確認** (`GET /auth/me`):
  - レスポンス: `csrf_token` なければ新規トークン発行（補填）

- **状態変更エンドポイント** (POST /articles/{id}/star など):
  - リクエスト時に `X-CSRF-Token` ヘッダが必須（`CSRF_PROTECTION_ENABLED=true` の場合）
  - 不一致 → 403

### 環境変数（`.env.example` に追記）

```
# CSRF protection (double-submit cookie)
CSRF_PROTECTION_ENABLED=false
CSRF_EXEMPT_PATHS=/auth/login
```

### テスト

- `backend/tests/test_csrf_middleware.py`: トークン生成・比較・ミドルウェア動作・免除条件・環境変数パース
- `backend/tests/test_api_auth.py` 追記: ログイン時の `csrf_token` Cookie 発行確認
- `web/src/lib/__tests__/api.test.ts`: Header 付与・BFF パススルー

### セキュリティ面での改善

- **CSRF 攻撃耐性**: Double-submit cookie + Bearer 免除で、ブラウザと API の両方を保護
- **タイミング攻撃耐性**: `hmac.compare_digest` で定時間比較
- **Cookie 盗聴耐性**: `Secure` フラグで HTTPS のみ配信（本番環境）

### 既存との互換性

- **iOS**: Bearer 認証で CSRF 免除。既存実装の変更不要
- **テスト**: `CSRF_PROTECTION_ENABLED=false` が既定。既存テスト互換性保証

### 残課題・トレードオフ

- **サブドメイン共有 Cookie**: メインドメイン上の悪意サブドメインが同一 Cookie を読める（既存 session_token と同じリスク）。サブドメイン間での CSRF 対策は本 ADR 対象外（信頼できるサブドメインのみ配置という前提）。
- **クライアント実装**: Web 側で `X-CSRF-Token` Header 付与が必須。実装忘れのリスク。BFF パススルーで単一箇所に集約し軽減。

## トレードオフと留保事項

### Double-submit の弱点：サブドメイン

悪意サブドメイン（例 `evil.example.com`）が同一メインドメイン Cookie を読める場合、csrf_token を窃取されて double-submit が破綻する。ただし：
- 本実装は信頼できるサブドメインのみ使用という前提
- session_token（httpOnly）も同じリスク
- Cookie の SameSite=Lax で関連コンテキストでのみ付与されることで、悪意サブドメイン → クロスサイト POST の CSRF 構成は成り立ちにくくなる

### 旧ブラウザ対応

SameSite 未サポートの旧ブラウザ（IE11 等）は double-submit に依存。本アプリは iOS・モダンブラウザ想定のため許容する（IE11 サポート不要）。

### iOS CSRF について

iOS は `WKWebView` でも独自 HTTP 実装でも Bearer トークンが標準で、CSRF 攻撃シナリオが成り立たない（トークン詐称・盗聴は別のセキュリティ層）。double-submit cookie は iOS に無関係で、Bearer 免除で対応。

