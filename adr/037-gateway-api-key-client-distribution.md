# ADR-037: ゲートウェイ API キー（X-API-Key）のクライアント配布方法 — ユーザー入力の廃止

**ステータス:** 採用済み（移行中：web=Issue, iOS=Issue でランタイム切替を実装）
**決定日:** 2026-06-27
**対象読者:** フロントエンド（web / iOS）/ インフラ / 運用担当
**関連:** [ADR-001](001-web-bff-proxy.md)（Web BFF プロキシ）・[ADR-007](007-ios-direct-backend-access.md)（iOS は backend 直アクセス）・[ADR-013](013-session-auth-and-user-management.md)（セッション認証）・[デプロイ状況記録 §7](../operations/2026-06-11-deployment-status.md)

## 背景

基盤の `X-API-Key`（共有 API ゲート、[backend-design §6](../design/backend-design.md#6-認証ユーザー管理adr-013)）の目的は **「不正なクライアントが backend API を叩くのを防ぐ」基盤防御**であり、利用者ごとのログイン認証（セッション／Passkey、ADR-013 / ADR-035）とは別レイヤである。

しかし現状、正式クライアントが**この共有キーをエンドユーザーに入力させて**いる:

- **web**: 初回 SetupModal で backend URL と API キーを利用者が入力 → `localStorage`（`api_base_url` / `api_key`）に保存（ADR-001 の BFF はそれを転送するだけ）。
- **iOS**: 初回 InitialSetupView と設定画面で入力 → `UserDefaults` に保存。

これは設計目的と乖離している。共有ゲートキーは「クライアント実装の構成値」であって利用者が知る・入力するものではない。加えて web では `localStorage` 保管が XSS 露出面となり、利用者にキーが見えてしまう。

## 決定

**ゲートウェイ API キー（および backend URL）のユーザー入力を廃止し、クライアント種別ごとに構成値として供給する。** backend 側の `X-API-Key` 検証（`hmac.compare_digest`、共有キー）は**変更しない**。配布方法のみを変える。

- **web — BFF サーバーサイド env 注入**:
  BFF プロキシ（`web/app/api/backend/[...path]/route.ts`、ADR-001）が **Vercel のサーバー専用環境変数** `BACKEND_BASE_URL` / `BACKEND_API_KEY` を読み、backend へ `X-API-Key` を注入する。
  - **`NEXT_PUBLIC_` を付けない**（付けるとクライアントバンドルへ露出する）。
  - クライアント（ブラウザ）は `X-API-Key` / `X-Backend-Base-Url` を送らない。**API キーはブラウザに一切露出しない**。

- **iOS — ビルド時注入**:
  既存機構（`Secrets.xcconfig` → `Info.plist` → `Bundle`、ADR-008 の永続化とは別系統）に集約し、入力画面（InitialSetupView / 設定の API 設定欄）を撤去する。`AppState` は Info.plist 由来の注入値のみ参照する。

## 検討した選択肢

### (A) web のキー供給方法

| 選択肢 | メリット | デメリット |
|--------|---------|----------|
| ユーザー入力 + localStorage（現状） | 実装済み | 目的と乖離。XSS 露出面。利用者にキーが見える |
| `NEXT_PUBLIC_*` でクライアント埋め込み | 実装が小さい | **JS バンドルにキーが露出**し誰でも抽出可能。BFF が既にあるのに非推奨 |
| **BFF サーバー env 注入** | キーがブラウザに非露出。BFF 既存資産を活用 | サーバー env（Vercel）整備が必要 |

### (B) iOS のキー供給方法

| 選択肢 | メリット | デメリット |
|--------|---------|----------|
| ユーザー入力 + UserDefaults（現状） | 実装済み | 目的と乖離。利用者にキーが見える |
| **ビルド時注入（Secrets.xcconfig）** | 既存機構。入力 UI 不要 | キーはバイナリ同梱で抽出可能（下記）。配布物にキーが含まれる |

## セキュリティ上の注記（限界の明示）

- **iOS は backend へ直接通信（ADR-007、BFF を挟めない）**ため、共有キーは**アプリバイナリに同梱され抽出可能**である。直接通信である以上、クライアント側での真の秘匿は不可能であり、本キーの目的は「不正クライアントによる**大量・自動アクセスの抑止**」に留まる（強固な認証は利用者セッション／Passkey が担う）。この限界を許容のうえで採用する。
- **web はサーバー専用 env のためブラウザに露出しない**。両クライアントで露出面の前提が異なる点に留意する。
- 露出面を本質的に下げるには共有キーをやめ利用者セッションのみに寄せる将来案もあるが、基盤防御（未ログイン経路・`/auth/login` 等への素のアクセス抑止）として共有ゲートは当面維持する。

## 影響

- **web（`news-listen-web`）**: BFF を env 注入へ変更、クライアントのヘッダ送出と localStorage 構成・入力 UI（SetupModal / 設定の API 欄）を撤去。
- **iOS（`news-listen-ios`）**: 入力画面撤去、`AppState` を Info.plist 注入値のみ参照へ。
- **親リポ（`news-listen`）**: Vercel に `BACKEND_BASE_URL` / `BACKEND_API_KEY`（非公開）を設定。`.env.example` とデプロイ状況記録を更新。
- **backend（`news-listen-backend`）**: **影響なし**（検証ロジック・共有キーともに不変）。
- **一回限りの前提**: Vercel への env var 登録（`vercel env add BACKEND_BASE_URL/BACKEND_API_KEY` を Production / Preview、`NEXT_PUBLIC_` なし）。
