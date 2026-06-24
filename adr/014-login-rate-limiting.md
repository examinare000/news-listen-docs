# ADR-014: ログイン試行のレート制限

**ステータス:** 採用済み
**決定日:** 2026-06-24
**対象読者:** バックエンド / Web / iOS 開発者
**関連:** [ADR-013](013-session-auth-and-user-management.md)（セッション認証の詳細）・[backend issue #6](../../../issues/6)・`agent-rules/12-security-guidelines.md` §3

## 背景

ADR-013 でセッション認証（`POST /auth/login` エンドポイント）が導入され、パスワードハッシュ化（bcrypt）によって単発のクラック耐性は強化された。しかし bcrypt の計算コストの高さは**オンライン総当たり攻撃**を完全には止められない。攻撃者が自動化したスクリプトで大量のログイン試行を送信する場合、IP 単位・ユーザー名単位で試行回数を制限することが必須となる（`agent-rules/12-security-guidelines.md` §3 「IP/ユーザー単位のレート制限」）。

同時に、1〜5 人の小規模家族運用では追加インフラ（Redis）の導入・運用負荷は重く、既存スタック（Cloud Run + Firestore）のみで実現する必要がある。

## 検討した選択肢

| 選択肢 | メリット | デメリット |
|---|---|---|
| **Firestore カウンタ方式**（採用） | 既存スタックのみ。複数インスタンス・再デプロイをまたいでカウント一貫。トランザクショナルな原子更新が可能 | ログインごとに Firestore read/write が増える（低頻度ユースケースでは許容） |
| slowapi（Flask 標準） | 実装が簡単 | メモリ内カウント → インスタンス毎にリセット。複数インスタンスでは保護が分散。Cloud Run 再デプロイで全カウントが消失 |
| Redis（専用キャッシュ）  | 高速・スケーラブル | 新規インフラ・運用・課金が増加。1〜5 人規模には過剰。構成の複雑化 |

## 決定

- **Firestore カウンタ方式を採用**する。`loginAttempts/{key}` コレクションで試行回数を記録し、`POST /auth/login` 処理の前処理で原子的にチェック・更新する。
- **カウント対象は IP/username 各単位で独立監視**する。すなわち `key` は `ip:{ip_hash}`と `username:{username}` の2種類を持つ。どちらか一方が閾値を超過したら 429 を返し、その試行をカウントしない。
- **生 IP は保存しない**。クライアント IP を SHA-256 ハッシュ化してキーとする（`X-Forwarded-For` → 信頼できる最左 IP のハッシュ）。ハッシュ化により、Firestore 漏洩時も特定個人の IP アドレスが直接露出しない。詐称は可能だが、username 単位カウントとの多層防御で実用的な耐性を得る。
- **ロック機構**: 閾値超過時は `loginAttempts` ドキュメントに `locked_until` タイムスタンプを記録。一時的なロック状態（例 15 分）に移行。ロック中は `429 Too Many Requests` + `Retry-After` ヘッダを返す。これは計算負荷の高い bcrypt を回避し、早期段階で拒否する。
- **カウンタリセット**: ログイン成功時に対象ユーザーの `loginAttempts` ドキュメント両方（IP/username）を削除。失敗時は `count` をインクリメント・`window_start` を更新。
- **スキーマ**: `loginAttempts/{key}` ドキュメント = `{ "key": "ip:{hash}" or "username:{username}", "count": int, "window_start": timestamp, "locked_until": timestamp or null }`
- **パラメータ化**: 環境変数で閾値・期間を制御可能にする。
  - `LOGIN_RATELIMIT_MAX_ATTEMPTS`: 閾値（既定 5 回）
  - `LOGIN_RATELIMIT_WINDOW_SECONDS`: スライディングウィンドウ期間（既定 900 秒 = 15 分）
  - `LOGIN_RATELIMIT_LOCKOUT_SECONDS`: ロック継続時間（既定 900 秒 = 15 分）
  - `0` を設定することで機能全体を無効化可能（開発環境など）。
- **既存コード流用**: `shared/firestore.py` の `@firestore.transactional` パターン（既存 `try_acquire_job_lock` と同様）で原子的な read-modify-write を実装。Firestore 競合・衝突をドライバレベルで回避する。

## 理由

- Firestore トランザクションにより、複数インスタンス下でもカウンタの整合性が保証される（slowapi では不可）。再デプロイ時もデータは永続。
- IP ハッシュ化は詐称に対する完全な防御ではないが、username 単位カウントと並行することで、同一ユーザーへの集中攻撃（username が既知の場合）と、IP スキャニング（複数ユーザー試行、IP が未知の場合）の両方を検出できる。
- ロック中に bcrypt を実行しないため、CPU 負荷が削減され、DoS 耐性が向上する。
- 環境変数による無効化により、開発・テスト環境では制限を外せる。本番環境では厳格に機能する。

## 結果（影響）

- **スキーマ**: Firestore に新規コレクション `loginAttempts/{key}` を追加。ドキュメント構造は上記参照。TTL ポリシーを将来設定し、古いロック記録を自動削除可能にする基盤を用意。
- **API**: `api/routers/auth.py` の `POST /auth/login` 処理に前処理を挿入。
  - `check_login_rate_limit(ip_hash: str, username: str) -> bool` 関数を実装。制限中なら `HTTPException(429)` を発生。
  - ログイン成功時（パスワード正合性が確認後）に `clear_login_attempts(ip_hash: str, username: str)` を実行（トランザクション）。
  - 失敗時（ユーザー存在しない or パスワード不一致）は `loginAttempts` をインクリメント（トランザクション）。このため、ユーザー存在秘匿（ADR-013 §決定の「汎用 401」）と両立。
- **依存関数**: `api/dependencies.py` に `get_client_ip(request) -> str` を追加（`X-Forwarded-For` の解析）。その SHA-256 ハッシュを生成する `hash_ip(ip: str) -> str`。
- **新規環境変数**: `LOGIN_RATELIMIT_MAX_ATTEMPTS`・`LOGIN_RATELIMIT_WINDOW_SECONDS`・`LOGIN_RATELIMIT_LOCKOUT_SECONDS`（`.env.example` に追記）。
- **エラーハンドリング**: 429 レスポンスボディは既存エラー構造（`{ "detail": "..." }`）に従い、ユーザーには `"ログイン試行が多すぎます。しばらく後に再度お試しください。"` のような汎用メッセージを返す。内部ログには `[WARN] Rate limit triggered: ip_hash={hash}, username={username}, attempts={count}` を記録。
- **Web クライアント**: ローディング中の重複送信は既存 UI 側の `disabled` 属性で防止。429 レスポンスに対して Retry-After を読み、ユーザーに「時間をおいてからお試しください」と表示。
- **iOS クライアント**: 同様に送信ボタンの無効化と `Retry-After` の UI 反映（既存パターン）。

## トレードオフと留保事項

- **Firestore コスト**: ログインごとに read/write が1回増える。1 日 100 ログイン（5 ユーザー × 20 回）と見積もると月 3000 読み書きで、Firestore 無料枠内（50K 読み書き/日）は余裕。本番運用で上限を超えた場合は、Redis 導入を新 ADR で上書きする。
- **XFF 信頼**: `X-Forwarded-For` はクライアント由来のヘッダで詐称可能。ただし Cloud Run を通す場合、GCP のプロキシが信頼できる右側値を付与するため、実運用では左から $n$ 個を信頼する慣例が一般的。本実装では最左 1 個（直接接続元）とし、詐称の影響を username 単位レートリミットで緩和。
- **同一 IP からの複数ユーザー**: 家族 5 人が同一 WiFi IP で共有する環境では、1 人の失敗が他 4 人のカウントを消費する。閾値が 5 回であれば、通常ログインで影響は軽微（数日に 1 回ログイン）。必要に応じて閾値を上げるか、username 比重を高めることで対応。
- **スライディングウィンドウ実装の簡略化**: 本実装は `window_start` の時刻から `WINDOW_SECONDS` 経過したら自動的にカウント 0 へリセット（`check_login_rate_limit` で時間チェック）。より精密な「直近 N 秒のイベント数」ではなく、簡易版だが実務的には十分。

## 関連する今後の課題

- **セルフサービスパスワードリセット**: メール送信によるリセット機能を実装する場合、そのメール認証トークンの有効期限・レート制限も定義する（新 ADR）。
- **監査ログ**: ログイン失敗・異常な試行パターンをクエリ可能にする専用ログテーブルの追加 → [ADR-015](015-audit-logging.md) で採用済み。
- **多要素認証（MFA）**: 将来的に TOTP / メール OTP を導入する場合の設計。本 ADR でも`loginAttempts` データは流用可。
