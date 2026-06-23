# ADR-013: セッションベース認証とマルチユーザー管理（ログイン）の導入

**ステータス:** 採用済み
**決定日:** 2026-06-23
**対象読者:** バックエンド / Web / iOS 開発者
**関連:** [ADR-007](007-ios-direct-backend-access.md)（単一ユーザー前提を更新）・[ADR-012 (C)](012-featured-sites-and-onboarding.md)（admin API の認証方針を更新）・[PRD §5.4/§7.3/§8](../prd/2026-05-31-news-listen.md)・[backend-spec §1/§4.7](../spec/2026-06-10-backend-spec.md)・`agent-rules/12-security-guidelines.md`

## 背景

従来は **デプロイ単位の単一ユーザー前提**（`USER_ID`、ADR-007）で、全 API を **共有 `X-API-Key`** 一枚で保護していた。鍵を知る者は誰でも全データへアクセスでき、誰が操作したかの区別も無い。PRD ではマルチユーザー管理を「Firebase Auth による本格化」としてフェーズ2（P1）に置いていた。

実利用（家族・友人 1〜5 人での共有、PRD §3）に踏み込むにあたり、次が必要になった。

1. 利用者ごとのログインと本人識別（誰の Star／設定か）。
2. 管理者がユーザーを発行・停止できる管理機能。
3. これらを **既存の Cloud Run + Firestore 構成のまま**、Web（ブラウザ）と iOS（ネイティブ）双方で成立させること。

決定すべき論点は (A) 認証基盤の選定、(B) セッショントークンの保持・受け渡し方式、(C) 管理操作の権限分離、の3点。

## 検討した選択肢

### (A) 認証基盤

| 選択肢 | メリット | デメリット |
|---|---|---|
| Firebase Auth（当初プラン） | マネージドで OAuth/MFA まで拡張可 | SDK・トークン検証・課金体系が増える。1〜5 人規模には過剰。Firestore 直結の現構成へ別認証系を接ぎ木する複雑さ |
| 共有 `X-API-Key` 継続 | 追加ゼロ | 本人識別・権限分離が原理的に不可能。実利用の要件を満たせない |
| **自前セッション（bcrypt + 不透明トークン + Firestore）** | 既存スタックのみで完結。本人識別・admin ロール・強制ログアウトを完全制御。規模相応に最小 | パスワード/セッション管理を自前で負う（実装責任） |

### (B) トークンの保持・受け渡し

| 選択肢 | メリット | デメリット |
|---|---|---|
| JWT（自己完結トークン） | サーバ状態不要 | 失効が難しい（最後の admin 失効・PW リセット時の即時無効化要件と相性が悪い） |
| **不透明トークン + サーバ側 Session（Firestore）** | サーバ側で即時失効可能。DB には SHA-256 ハッシュのみ保存（生トークン非保存） | セッション照合で1リード増 |
| クライアント保持方式を Web/iOS で共通化 | — | Web は XSS、iOS は端末保管と脅威が異なり一律にできない |

### (C) 管理操作の権限分離

| 選択肢 | メリット | デメリット |
|---|---|---|
| `X-API-Key` のまま（ADR-012 (C)） | 追加ゼロ | ユーザー CRUD を鍵保持者全員ができてしまう |
| **`role` による admin/user 分離（`require_admin`）** | ユーザー管理を admin に限定。自己ロックアウトも防げる | ロール判定とガードの実装が必要 |

## 決定

- **(A) 自前セッション認証を採用**する。Firebase Auth は見送り、`shared/security.py`（bcrypt によるパスワードハッシュ、`secrets` による不透明トークン生成）+ Firestore の `users/{username}` / `sessions/{session_id}` で構成する。PRD のフェーズ2項目「Firebase Auth によるマルチユーザー管理」は **本 ADR の独自方式で実装済み**に置き換える。
- **(B) サーバ側 Session + デュアルトランスポート**。生トークンは **SHA-256 ハッシュ化して Session に保存**し、照合は毎回ハッシュ比較。受け渡しは **Web=httpOnly Cookie `nl_session`**（BFF が中継）、**iOS=`Authorization: Bearer`**（Keychain 保管）の二系統を、バックエンドの認証依存が「Bearer ヘッダ → Cookie」の順で吸収する。TTL は `SESSION_TTL_HOURS`（既定 168h=7日）。
- **(C) `role`（`admin` / `user`）で権限分離**。ユーザー CRUD（`/admin/users`）は `require_admin` で保護する。一方、おすすめサイト管理（`/admin/featured-sites`、ADR-012）は当面 `X-API-Key` のまま据え置く（運用者専用の位置づけは不変）。
- **自己ロックアウト・運用事故の防止**として、(1) 最後の admin の降格・削除は `HTTP 409` で拒否、(2) **降格・パスワードリセット・削除の各操作で対象ユーザーの全セッションを失効**（旧権限・旧資格情報での継続アクセスを即時遮断）、(3) Web/iOS の管理画面は**自分自身**のロール変更・削除ボタンを非表示にする。
- ログイン失敗は **ユーザー存在の有無を伏せた汎用メッセージ**（`401`）で返す。

## 理由

- 1〜5 人規模で必要なのは「ログイン・本人識別・admin によるユーザー発行」であり、これは既存スタック内の自前セッションで過不足なく満たせる。Firebase Auth の運用・課金・SDK 増を負う便益が小さい。
- 不透明トークン + サーバ側 Session は、**最後の admin 保護・PW リセット時の即時失効**という要件と素直に噛み合う（JWT では困難）。DB へは生トークンを残さずハッシュのみ保存することで、Firestore 漏洩時もセッション乗っ取りを防ぐ。
- Web と iOS は脅威モデルが異なる（XSS vs 端末保管）。httpOnly Cookie（JS から読めない）と Keychain（端末ロック連動・バックアップ外）をそれぞれの最適解とし、サーバ側で受け口を二系統に開くことで、クライアント実装の自由度と安全性を両立する。

## 結果（影響）

- **スキーマ**: `shared/models.py` に `User`（`username` / `user_id` / `password_hash` / `role` / `display_name` / 監査タイムスタンプ）と `Session`（`session_id`=トークンの SHA-256 ハッシュ / `user_id` / `username`・`role` のキャッシュ / `expires_at`）を追加。Firestore コレクション `users/{username}`・`sessions/{session_id}` を新設。
- **API**: `api/routers/auth.py`（`POST /auth/login`・`POST /auth/logout`・`GET/PATCH /auth/me`・`POST /auth/password`）と、`api/routers/admin.py` のユーザー CRUD（`GET/POST /admin/users`・`PATCH/DELETE /admin/users/{username}`、`require_admin`）を追加。`api/dependencies.py` に `get_current_user` / `get_user_id` / `require_admin` を実装し、`get_user_id` は **環境変数 `USER_ID` 固定からログインセッション由来へ**切り替えた（ADR-007 の単一ユーザー前提を更新）。
- **Web**: `AuthContext` がセッション状態（`unknown`/`authenticated`/`unauthenticated`）を `GET /auth/me` で解決。入口ゲート `app/page.tsx` は未認証なら `LoginModal`、設定画面に `AccountSection`（表示名・PW 変更・ログアウト）、`/admin/users` に管理画面を追加。**BFF プロキシは `Cookie` 中継と `Set-Cookie` の通し**を行う（[web-spec §6](../spec/2026-06-10-web-frontend-spec.md)）。
- **iOS**: `SessionStore`（Keychain `kSecAttrAccessibleAfterFirstUnlock`）にトークンを保管し、`APIClient` が `Authorization: Bearer` で送出。起動ゲートで `refreshAuth()`→未認証なら `LoginView`、設定に `AccountSettingsView`、`AdminUsersView` を追加。
- **初期データ**: `backend/scripts/seed_users.py` が環境変数（`INITIAL_ADMIN_USERNAME`/`INITIAL_ADMIN_PASS`・`INITIAL_USER_USERNAME`/`INITIAL_USER_PASSWORD`）から初期ユーザーを冪等投入。**初回デプロイ後に必ず既定パスワードを変更**すること。
- **新規環境変数**: `SESSION_TTL_HOURS`（既定 168）・`SESSION_COOKIE_SECURE`（既定 true、ローカル開発は false）・上記初期ユーザー4変数（`.env.example` 参照）。
- **残課題**: パスワードリセットのセルフサービス（メール送信）は本 ADR の範囲外。監査ログは本 ADR の範囲外。ログイン試行のレートリミットは [ADR-014](014-login-rate-limiting.md) で解決済み。
