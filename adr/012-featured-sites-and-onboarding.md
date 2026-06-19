# ADR-012: おすすめサイトの配信（featuredSites）と初回オンボーディングのサーバ永続化

**ステータス:** 採用済み
**決定日:** 2026-06-19
**対象読者:** バックエンド / Web / iOS 開発者
**関連:** [ADR-007](007-ios-direct-backend-access.md)（単一ユーザー前提）・[backend-spec §4.5/§4.5.1](../spec/2026-06-10-backend-spec.md)・`agent-rules/12-security-guidelines.md`

## 背景

RSS ソースは「名前 + URL を直接入力」してのみ購読でき、新規ユーザーは初回ログイン後に空のフィードへ直面していた。
最初の購読を促すため、**システム提供のおすすめサイトをワンクリックで購読できる UI**（Web `/subscriptions`・iOS Settings・初回ステップ）を追加する。これに伴い次の設計判断が必要になった。

1. おすすめサイトのデータをどこで管理するか。
2. 「初回かどうか」をどこに永続化するか。
3. 管理用 CRUD API の認証をどうするか。

バックエンドは Firestore（スキーマレス）、認証は共有 `X-API-Key`、デプロイ単位で単一ユーザー（`USER_ID`、ADR-007）。

## 検討した選択肢

### (A) おすすめサイトの管理方法

| 選択肢 | メリット | デメリット |
|---|---|---|
| クライアントにハードコード（現状） | 実装ゼロ | 変更のたびに全クライアント再デプロイ。サムネイル等の表現も持てない |
| **Firestore `featuredSites` コレクション + 管理 CRUD API** | サーバ側で追加・更新・削除でき全クライアントへ即反映。サムネイル/説明/表示順を保持 | コレクションと管理 API の追加が必要 |

### (B) 初回判定の永続化先

| 選択肢 | メリット | デメリット |
|---|---|---|
| クライアントローカル（localStorage / UserDefaults） | バックエンド変更不要 | 端末・ブラウザごとに初回ステップが再発。Web/iOS 間で共有されない |
| **`UserPrefs.onboarding_completed`（Firestore）** | 単一ユーザー前提のため Web/iOS で共有。一度完了すれば再表示されない | UserPrefs にフィールド追加が必要 |

### (C) 管理 API の認証

| 選択肢 | メリット | デメリット |
|---|---|---|
| **共有 `X-API-Key`（既存 verify_api_key）** | 追加インフラ不要。単一ユーザー運用と整合 | 通常クライアントと同じ鍵で管理操作も可能になる |
| `ADMIN_API_KEY` を分離 | 管理操作を別権限に隔離 | 鍵管理が増える。現状の単一運用者では過剰 |

## 決定

- (A) おすすめサイトは Firestore コレクション **`featuredSites/{id}`**（グローバル・ユーザー横断、`id` は name から生成した slug）で管理し、`order` 昇順で配信する。参照は `GET /settings/featured-sources`、管理は `api/routers/admin.py` の `GET/POST/PUT/DELETE /admin/featured-sites`。
- (B) 初回判定は **`UserPrefs.onboarding_completed: bool = False`** に永続化する。`GET /settings/onboarding` で取得、`POST /settings/onboarding/complete` で完了化（`add_source` と同じ全置換更新）。
- (C) 管理 API は当面 **共有 `X-API-Key`** で保護する。`ADMIN_API_KEY` 分離は将来の拡張余地として `admin.py` 冒頭にコメントで残す。

## 理由

- サーバ集中管理によりクライアント再デプロイなしでおすすめを更新でき、表現（サムネイル・説明・順序）も持てる。
- 単一ユーザー前提では UserPrefs にフラグを持たせるだけで Web/iOS 双方の初回体験を一度で済ませられる。
- 運用者が 1 人である以上、管理 API を別鍵で隔離する便益は小さく、既存認証で十分。

## 結果（影響）

- スキーマ: `shared/models.py` に `FeaturedSite` と `UserPrefs.onboarding_completed` を追加（既存ドキュメントは default で後方互換）。
- 永続化更新は `merge` ではなく全置換 `.set()`（`save_user_prefs` 既存挙動）を用い、required な `default_difficulty` の欠落を防ぐ。
- ワンクリック購読は新 API を足さず既存 `POST /settings/sources`（`addSource`）を再利用する。重複は `409` を「登録済み」として扱う。
- 初回ステップ UI は Web=`app/page.tsx` のリダイレクト前モーダル、iOS=`ContentView` 上の `fullScreenCover`（起動をブロックしない）で提示する。URL 直接入力の購読導線は従来どおり残す。
- 初期データは `backend/scripts/seed_featured_sites.py` で投入（DB 管理の正は管理 API）。
- 管理 API を別権限にする必要が生じた場合は `ADMIN_API_KEY` 分離を新 ADR で上書きする。
