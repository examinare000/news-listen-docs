# ADR-015: 監査ログ（admin 操作・ログインイベント）

**ステータス:** 採用済み
**決定日:** 2026-06-24
**対象読者:** バックエンド / 運用担当者
**関連:** [ADR-013](013-session-auth-and-user-management.md)（セッション認証・ユーザー管理）・[ADR-014](014-login-rate-limiting.md)（レートリミット・基盤共有）・[backend issue #8](../../../issues/8)・`agent-rules/12-security-guidelines.md`

> **補足（2026-06-25・issue #42）:** 本 ADR の `AuditLogger` 基盤を**利用者のデータ変更操作**へ拡張した（当初は auth/admin のみ計装）。追加 `AuditAction`: `article_star` / `article_dismiss` / `rss_source_add` / `rss_source_remove` / `preferences_update` / `onboarding_complete`（issue #44 で `article_mark_read`、issue #47 で `storage_cleanup` も追加）。`api/routers/articles.py`・`api/routers/settings.py` の各書き込みエンドポイントに `record(...)` を計装（actor は `get_current_user` の Session、DB 変更成功後に記録し 404/409 等のエラー時は記録しない）。`details` は対象識別子のみ（article_id・RSS url/name）で、**プリファレンス更新は変更フィールド名のみ・値は記録しない**（機微情報非記録の方針を踏襲）。基盤・記録項目・ベストエフォート・改竄防止の決定本文は不変。

## 背景

ADR-013 でマルチユーザー化（セッション認証・admin によるユーザー管理）が導入された結果、「誰がいつ何をしたか」が初めて意味を持つようになった。しかし現状、admin のロール変更・パスワードリセット・ユーザー削除は事後追跡が不可能で、ログイン失敗は一時的な `warning` ログが出るだけ・永続的な監査記録は存在しない。これは運用事故や内部不正を検知できないセキュリティギャップである。

ADR-014（レートリミット）が「監査ログ」を今後の課題として挙げていた通り、認証・レートリミット導入直後の今、追跡基盤を整えるべきタイミングである。レートリミットと同じ既存スタック（Cloud Run + Firestore）で実現でき、追加インフラは不要。

## 検討した選択肢

| 選択肢 | メリット | デメリット |
|---|---|---|
| **Firestore `auditLogs` コレクション + サービス層集約**（採用） | 既存スタックのみ。永続・クエリ可能。記録判断を1箇所に集約しテスト容易 | ログイン・admin 操作ごとに write が増える（低頻度ユースケースでは許容） |
| 各ルータに直書き | 追加の抽象なし | try/except・PII 除外・記録判断が各所に重複。計装漏れ・例外伝播事故のリスク |
| Cloud Logging（構造化ログ）のみ | 書き込み先の運用が不要 | 保持期間・クエリ・改竄防止の制御が弱い。admin への参照 API を別途要する |

## 決定

- **Firestore `auditLogs` コレクションを新設**し、認証イベントと admin 操作を追記する。doc-id は `add()` による自動採番（追記専用・直引き不要のため衝突回避を優先。他コレクションの `set(doc-id)` 流儀とは意図的に異なる）。
- **記録の意思決定をサービス層 `AuditLogger`（`api/audit.py`）に集約**する。各ルータは `record(action, actor, target_username, ip, details)` を1行呼ぶだけにする。依存方向は ルータ → AuditLogger → FirestoreClient → Firestore（内向き）。
- **記録対象**: `login_success` / `login_failure` / `login_lockout` / `logout` / `user_create` / `user_update` / `user_role_change` / `user_password_reset` / `user_delete` / `session_revoke`（`action` は Literal 列挙で網羅を担保）。
- **記録項目**: action・timestamp・actor（user_id / username。未認証ログイン失敗時は null）・target_username・ip・details。**平文パスワード・生トークン・password_hash は記録しない**（型と計装テストの二重防御）。
- **IP は生値で保存**する。フォレンジック（実際の発信元 IP による追跡）が監査の目的であり、参照は admin 限定のため。ADR-014 のレートリミットが IP を SHA-256 ハッシュ化するのとは方針が異なる（用途が「同一性判定」ではなく「追跡」のため）。`api/dependencies.py` の `get_client_ip` の値をそのまま記録する。
- **ベストエフォート書き込み**: 監査ログの書き込み失敗で本操作（ログイン・CRUD）を失敗させない。例外捕捉と error ログ化の境界を `AuditLogger.record` の1箇所に閉じ込める。`FirestoreClient.append_audit_log` は例外を握り潰さず伝播させ、失敗の可観測性（error ログ）を確保する。時刻は `clock` を注入可能にし、テストで固定する。
- **改竄防止（追記専用）**: `FirestoreClient` に監査ログの更新・削除メソッドを設けない。
- **参照 API**: `GET /admin/audit-logs`（`require_admin`）を追加。`action` フィルタと `limit`（既定 50・範囲 1〜500）で timestamp 降順に取得する。レスポンスに `actor_user_id`（内部 UUID）は含めない。
- **インデックス**: `action` フィルタ + `timestamp` 降順は複合インデックスを要するため、`backend/firestore.indexes.json` に `auditLogs`（action ASC + timestamp DESC）を追加する。`actor` / `target` フィルタは v1 では提供しない（YAGNI。必要時に追加）。

## 理由

- 記録判断を `AuditLogger` 1箇所に集約することで、各ルータの計装が1行で済み、ベストエフォートの try/except や PII 除外の重複・漏れを構造的に防げる。clock 注入と DB 注入により各層を単独でテストできる。
- 生 IP 保存は、監査の目的が「個人の同一性判定」ではなく「事故・不正の事後追跡」であり、参照が admin に限定されることから妥当。レートリミット（漏洩時の個人特定リスクを下げるためハッシュ化）とは要件が異なる。
- ベストエフォートにより、Firestore 一時障害時でも認証・管理操作の可用性を損なわない。一方で書き込み失敗を error ログに残すことで可観測性を確保し、監査の盲点を最小化する。
- 追記専用・更新削除 API 不在により、監査ログの改竄耐性を構造的に高める。

## 結果（影響）

- **スキーマ**: Firestore に新規コレクション `auditLogs`（自動採番）を追加。フィールドは [backend 設計書 §9](../design/backend-design.md#auditlogs-コレクションadr-015) を参照。
- **モデル**: `shared/models.py` に `AuditAction`（Literal）と `AuditLog`（BaseModel）を追加。
- **永続化**: `shared/firestore_client.py` に `append_audit_log`（add で追記）/ `list_audit_logs`（timestamp DESC・任意 action フィルタ）を追加。
- **サービス層**: `api/audit.py` に `AuditLogger` を新設。`api/dependencies.py` に `get_audit_logger` を追加。
- **計装**: `api/routers/auth.py`（login 成功/失敗/ロックアウト・logout）と `api/routers/admin.py`（ユーザー CRUD・ロール変更・PW リセット・セッション失効）に `record(...)` 呼び出しを挿入。admin の各エンドポイントは actor 取得のため `actor: Session = Depends(require_admin)` と `Request` を引数注入する（認可は等価で維持）。
- **API**: `GET /admin/audit-logs` を追加（`AuditLogResponse` / `AuditLogsResponse`）。
- **インデックス**: `backend/firestore.indexes.json` に `auditLogs`（action ASC + timestamp DESC）を追加。

## トレードオフと留保事項

- **Firestore コスト**: ログイン・admin 操作ごとに write が1回増える。小規模運用（1〜5 人）では無料枠内に十分収まる。大量化したら TTL ポリシーやエクスポート（BigQuery）を新 ADR で検討する。
- **生 IP の保存**: プライバシー観点では生値保存はハッシュ化より露出リスクが高い。参照を admin に限定し、Firestore 漏洩時のリスクを運用（アクセス制御）で抑える前提とする。要件が変われば再検討する。
- **details の自由度**: `details` は任意の dict であり、型だけでは機微情報の混入を完全には防げない。計装テストで「PW・トークンが details に入らない」を behavior として固定し二重防御する。
- **ベストエフォートの監視盲点**: 監査書き込み失敗は error ログに残るのみ。運用は error ログ監視に委ねる。
- **参照フィルタの限定**: v1 は `action` フィルタのみ。`actor` / `target` での絞り込みは複合インデックス追加を伴うため将来課題とする。

## 関連する今後の課題

- **保持・エクスポート**: 監査ログの保持期間ポリシー（TTL）と BigQuery 等への長期エクスポート。
- **参照フィルタ拡張**: `actor` / `target` / 期間によるフィルタと対応インデックス。
- **アラート**: 異常パターン（短時間の大量 `login_failure` / `user_delete` など）の検知・通知。
