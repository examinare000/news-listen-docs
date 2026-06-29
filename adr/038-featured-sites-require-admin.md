# ADR-038: おすすめサイト管理 API を require_admin（role ベース）で保護する

**ステータス:** 採用済み
**決定日:** 2026-06-29
**対象読者:** バックエンド / Web 開発者・インフラ担当
**関連:** [ADR-012](012-featured-sites-and-onboarding.md)（おすすめサイト配信・本 ADR が (C) を上書き）・[ADR-013](013-session-auth-and-user-management.md)（セッション認証と role・本 ADR が (C) のおすすめサイト据え置きを上書き）・[ADR-015](015-audit-logging.md)・[backend 設計書（Admin）](../design/backend-design.md)

## 背景

おすすめサイト CRUD `/admin/featured-sites`（`GET/POST/PUT/DELETE`）は、[ADR-012(C)](012-featured-sites-and-onboarding.md) の決定により共有 `X-API-Key`（API ゲート）のみで保護されていた。その後 [ADR-013](013-session-auth-and-user-management.md) で `role`（`admin`/`user`）と `require_admin` が導入され、ユーザー管理 `/admin/users`・監査ログ `/admin/audit-logs` は role ベースで保護されたが、おすすめサイトは ADR-013(C) で「当面 `X-API-Key` のまま据え置く」とされ、**認可ポリシーが非対称**のまま残っていた。

`featuredSites` はグローバル（ユーザー横断）のプリセットであり、共有 `X-API-Key`（BFF ゲートウェイキー）を経由できれば、**非 admin のログインユーザーでも全ユーザーに影響するおすすめサイトを作成・更新・削除し得る**。Web 側に管理者向けおすすめサイト管理画面（[web #36](../design/web-design.md)）を追加するにあたり、その認可前提として backend 側の role ベース保護が必要になった。

## 検討した選択肢

| 選択肢 | メリット | デメリット |
|---|---|---|
| 現状維持（`X-API-Key` のみ） | 追加ゼロ | 非 admin が全ユーザー横断のおすすめサイトを改変し得る。`/admin/users` と非対称 |
| `ADMIN_API_KEY` を分離（ADR-012 が将来案として記載） | 鍵で管理操作を隔離 | 鍵管理が増える。既に `role`/`require_admin` 基盤がある今は冗長。ログインユーザー単位の監査と結びつかない |
| **`require_admin`（role ベース）を適用** | `/admin/users`・`/admin/audit-logs` と同一ポリシーに統一。既存 ADR-013 の基盤を再利用し追加インフラ不要。操作主体をセッション＝ユーザーで識別できる | 既存テストの認可前提を admin セッションへ更新する必要がある |

## 決定

`/admin/featured-sites` の **4 エンドポイント全て（`GET` 一覧を含む）** に `Depends(require_admin)` を適用し、共有 `X-API-Key`（API ゲート）に加えて `role=="admin"` を要求する。`/admin/users`・`/admin/audit-logs` と同一の認可ポリシーに揃える。

- 非 admin（`role!="admin"`）は `403`、未ログイン（セッション無し）は `401`。依存解決は本体ロジック（404/409 判定）より前に走るため、非 admin はリソース存在を推定できない。
- 一覧（`GET`）も admin 限定とする。公開のおすすめ取得は別経路 `GET /settings/featured-sources` が担うため、管理用 `GET /admin/featured-sites` を admin 限定にしても公開機能に影響しない。
- 監査ログ（ADR-015）の付与はスコープ外（本 ADR では認可ガードのみ）。
- これにより [ADR-012(C)](012-featured-sites-and-onboarding.md) の「`X-API-Key` のまま据え置き／`ADMIN_API_KEY` 分離は将来案」と [ADR-013(C)](013-session-auth-and-user-management.md) の「おすすめサイトは当面 `X-API-Key` のまま」を**上書き**する。`ADMIN_API_KEY` 分離案は role ベース採用により不要となり破棄する。

## 理由

`role`/`require_admin` の基盤は ADR-013 で既に導入済みであり、これを再利用すれば追加インフラ無しでポリシーを統一できる。`ADMIN_API_KEY` 分離は鍵管理を増やすうえ、操作主体をログインユーザーで識別できず、将来の監査ログ（ADR-015）とも噛み合わない。グローバルなおすすめサイトを全ユーザーに影響する形で改変できる権限は、運用者（admin）に限定するのが妥当。

## 結果（影響）

- `api/routers/admin.py` の featured-sites 4 エンドポイントに `dependencies=[Depends(require_admin)]` を付与。冒頭 docstring と `api/main.py` の include コメントを role ベース保護に更新（陳腐化解消）。
- 既存テストを admin セッション前提に更新し、非 admin `403`・未認証 `401` のテストを追加（TDD）。
- Web は管理画面（`/admin/featured-sites`）＋設定導線＋`lib/api` の CRUD を追加し、クライアント（`role==='admin'`）＋ backend（`require_admin`）で二重防御する。
- `/admin/users`・`/admin/audit-logs`・`/admin/featured-sites` の 3 グループが同一の role ベース認可で保護され、ポリシーが対称化した。
