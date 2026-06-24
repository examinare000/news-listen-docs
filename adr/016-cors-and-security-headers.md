# ADR-016: CORS 設定とセキュリティレスポンスヘッダの導入

**ステータス:** 採用済み
**決定日:** 2026-06-24
**対象読者:** バックエンド / Web 開発者
**関連:** [ADR-001](001-web-bff-proxy.md)（BFF プロキシ／本 ADR で CORS を前進）・[ADR-013](013-session-auth-and-user-management.md)（httpOnly Cookie 認証の防御を補完）・[backend 設計書（セキュリティ）](../design/backend-design.md#7-セキュリティssrf--レートリミット)・`agent-rules/12-security-guidelines.md`・backend PR #18（issue #9）

## 背景

バックンド API（FastAPI、`api/main.py`）には **CORS 設定とセキュリティレスポンスヘッダがいずれも未設定**だった。

- CORS 未設定は、ブラウザから別オリジンの直接アクセスを成立させられないため、[ADR-001](001-web-bff-proxy.md) の **BFF プロキシ撤去（Web 残課題）のブロッカー**になっていた。ADR-001 では「フェーズ2でバックエンドに CORS を追加する」と留保していた。
- セキュリティヘッダ（HSTS / CSP / X-Frame-Options / X-Content-Type-Options 等）の不在は、httpOnly Cookie 認証（ADR-013）の防御を未完成のままにしていた。Cookie を JS から読めなくしても、クリックジャッキングや MIME スニッフィングといった周辺攻撃面が残る。

1 つのミドルウェア層を追加するだけで、両者を同時に前進させられる（一石二鳥）。決定すべき論点は (A) CORS の許可方針、(B) セキュリティヘッダの内容と適用方式、の 2 点。

## 検討した選択肢

### (A) CORS 許可方針

| 選択肢 | メリット | デメリット |
|---|---|---|
| `allow_origins=["*"]` | 設定不要 | 認証情報付き（credentials）リクエストと**併用不可**。任意オリジンからの資格情報送信を許す重大なリスク |
| **環境変数で許可オリジンを明示列挙（採用）** | 環境ごとに最小許可。`credentials` 対応。未設定時は全拒否（安全側） | 環境変数の設定が必要 |

### (B) セキュリティヘッダの適用方式

| 選択肢 | メリット | デメリット |
|---|---|---|
| 各レスポンスで個別付与 | 局所的 | 付け忘れが起きる。横断的関心事をルータに散らす |
| `BaseHTTPMiddleware` 派生 | 実装が簡単 | ストリーミング/バックグラウンドタスクとの相互作用に既知の落とし穴・オーバーヘッド |
| **素の ASGI ミドルウェア（採用）** | 全レスポンス（プリフライト含む）に確実に付与。副作用が明示的で軽量。ポリシー（純粋関数）と付与（薄いシェル）を分離しテスト容易 | ASGI の send ラップを自前で書く |

## 決定

- **(A) CORS は許可オリジンを環境変数 `CORS_ALLOWED_ORIGINS`（カンマ区切り）で明示列挙**する。未設定・空は **`[]`（全拒否）= 安全側デフォルト**。`allow_credentials=True` とし、`*` は使用しない。`allow_methods` は実使用に合わせ `GET/POST/PUT/PATCH/DELETE/OPTIONS`（`PATCH`/`PUT` は `PATCH /auth/me`・`PUT /admin/featured-sites/{id}` 等で実際に使用中のため必須）、`allow_headers` は `X-API-Key`（基盤ゲート）・`Authorization`（Bearer）・`Content-Type`。許可オリジン解析は純粋関数 `build_cors_options(env)`（`api/cors_config.py`）に集約する。
- **(B) セキュリティヘッダは素の ASGI ミドルウェア `SecurityHeadersMiddleware`（`api/middleware/security_headers.py`）で全レスポンスに付与**する。付与ヘッダは純粋関数 `build_security_headers(env)` が構築し、`http.response.start` 時に**既存ヘッダ名がある場合は上書きしない**（CORS ヘッダを潰さない）。
  - `Content-Security-Policy`（既定 `default-src 'none'; frame-ancestors 'none'` = API 向けの厳格値）
  - `X-Frame-Options: DENY`（クリックジャッキング対策）
  - `X-Content-Type-Options: nosniff`（MIME スニッフィング無効化）
  - `Referrer-Policy: no-referrer`
  - `Strict-Transport-Security`（HSTS）は **`SECURITY_HSTS_ENABLED` が真のときのみ付与**（既定無効。ローカル HTTP 開発で誤って pin させないため）。値は `max-age={SECURITY_HSTS_MAX_AGE}; includeSubDomains`。
  - 各ヘッダ値は環境変数で上書き可能（CSP/X-Frame-Options 等）だが、既定値で安全側に倒す。
- **ミドルウェア適用順序**: `api/main.py` で **CORS を先に `add_middleware`（内側）、SecurityHeaders を後に `add_middleware`（外側）**。Starlette は逆順適用のため、SecurityHeaders が最外層となり、CORS が生成するプリフライト（`OPTIONS`）応答にもセキュリティヘッダが乗る。

## 理由

- `*` + credentials はブラウザ仕様上両立せず、かつ実運用でも危険。許可オリジンの明示列挙＋未設定全拒否は、設定漏れがあっても「開きっぱなし」にならない安全側デフォルトを与える。
- ポリシー（純粋関数）と付与（ASGI シェル）の分離により、ヘッダ方針は `TestClient` を介さず単体テストでき、ミドルウェア自体はヘッダ辞書を注入されるだけの薄いシェルになる（テスト容易性・単一責務）。
- HSTS をフラグで明示有効化するのは、HTTP のローカル開発でブラウザにドメインを誤って固定（pin）させ、後続の開発を壊さないため。本番 HTTPS でのみ有効化する運用を既定にできる。
- SecurityHeaders を最外層に置くことで、CORS のプリフライト短絡応答を含む**全レスポンス**に確実にヘッダが乗る。「既存名は上書きしない」規則で CORS ヘッダとの干渉も避ける。

## 結果（影響）

- **新規モジュール**: `api/cors_config.py`（`build_cors_options`）、`api/middleware/security_headers.py`（`build_security_headers` + `SecurityHeadersMiddleware`）、`api/middleware/__init__.py`。
- **API**: `api/main.py` で `app = FastAPI(...)` 直後に CORS→SecurityHeaders の順でミドルウェアを登録。ルータ登録・`X-API-Key` 検証フローは不変（回帰なし）。
- **新規環境変数**（`.env.example` に追記）:

  | 変数 | 既定 | 説明 |
  |-----|-----|------|
  | `CORS_ALLOWED_ORIGINS` | 空（=全拒否） | 許可オリジンのカンマ区切り。`*` 不可 |
  | `SECURITY_CSP` | `default-src 'none'; frame-ancestors 'none'` | Content-Security-Policy |
  | `SECURITY_X_FRAME_OPTIONS` | `DENY` | X-Frame-Options |
  | `SECURITY_X_CONTENT_TYPE_OPTIONS` | `nosniff` | X-Content-Type-Options |
  | `SECURITY_REFERRER_POLICY` | `no-referrer` | Referrer-Policy |
  | `SECURITY_HSTS_ENABLED` | `false` | HSTS を有効化するか（本番 HTTPS のみ true） |
  | `SECURITY_HSTS_MAX_AGE` | `31536000` | HSTS の max-age 秒（有効時のみ適用） |

- **テスト**: `tests/test_api_cors.py`・`tests/test_api_security_headers.py`（許可/非許可オリジンのプリフライト、ヘッダの存在と既定値、env トグル、プリフライトでの CORS とセキュリティヘッダ共存、`X-API-Key` 認証の回帰）。
- **ADR-001 への影響**: バックエンド CORS が用意されたことで、Web の **BFF プロキシ撤去**（ADR-001 のフェーズ2残課題）に着手可能になる。撤去自体は Web 側の別タスク。
- **残課題**: グローバルなレート制限（ログイン以外、[ADR-014](014-login-rate-limiting.md) の範囲外）は本 ADR の対象外。許可メソッド/ヘッダの env 化は当面しない（実使用に追従、必要時に最小変更）。
