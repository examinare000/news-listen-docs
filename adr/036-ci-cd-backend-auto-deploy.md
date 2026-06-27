# ADR-036: backend submodule ポインタ更新時の本番自動デプロイ（CI/CD）

**ステータス:** 採用済み
**決定日:** 2026-06-27
**対象読者:** バックエンド / インフラ / 運用担当
**関連:** [デプロイ状況記録](../operations/2026-06-11-deployment-status.md)・[ADR-005](005-job-trigger-on-action.md)（ジョブ起動）・[ADR-013](013-session-auth-and-user-management.md)（セッション認証）

## 背景

本プロジェクトはマルチレポ（git submodule）構成で、`backend` の実装は別リポ（`news-listen-backend`）にある。親リポ `news-listen` は backend の特定コミットを **gitlink（submodule ポインタ）** で固定し、機能完成ごとに PR で「構成: … submodule ポインタを更新」として main に取り込んでいる。

しかし本番 backend（Cloud Run）の再デプロイは手動 `infra/deploy.sh` 任せで、**ポインタ更新と本番反映が自動連動していなかった**。

2026-06-27 のデプロイ状況確認で、この欠落が実害として顕在化した（[デプロイ状況記録 §7.6 / §8](../operations/2026-06-11-deployment-status.md)）:

- 稼働中 backend は認証導入（ADR-013, 2026-06-23）より**前**のコミット（`6b96a5c`, 2026-06-11）のまま取り残されていた。
- 一方フロント（Vercel）は最新で、`/auth/login` 等を要求 → **版ズレでログイン不能＝アプリが機能しない**状態だった。

「ポインタ更新 = 本番反映」を仕組みで保証し、この版ズレ事故を再発防止する必要がある。

## 決定

親リポ `examinare000/news-listen` の **main で backend gitlink が更新されたら、本番 Cloud Run の backend を自動デプロイ**する GitHub Actions ワークフロー（`.github/workflows/deploy-backend.yml`）を新設する。実デプロイは既存 `infra/deploy.sh` を**再利用**する（単一の真実）。

主要な設計判断:

- **GCP 認証は Workload Identity Federation（WIF, キーレス）**: 長期 SA 鍵を GitHub に置かず、OIDC の短命トークンで `news-listen-sa` をインパーソネートする。`assertion.repository` クレームで本リポに限定（最小権限）。
- **private submodule の checkout は fine-grained PAT**: default `GITHUB_TOKEN` は親リポしかアクセスできず private な backend を取得できないため、親+backend の `contents:read` を持つ PAT を用いる。
- **発火は完全自動**: backend ポインタ更新が main に入った時点で即デプロイ（承認ゲートなし）。
- **多層防御で版ズレを根絶**:
  1. `paths` フィルタ（`backend` gitlink 変更）で起動
  2. **冪等ガード**: pin SHA と稼働中リビジョンの `BACKEND_SHA` env を比較し、差分時のみデプロイ（`paths` 取りこぼし・手動再実行・重複起動への保険）
  3. **smoke test**: デプロイ後に `/health` 200 と `/openapi.json` に最新版固有パス（`/auth/login`・`/auth/passkey/login/verify`）が存在することを検証し、版ズレを検知して fail させる
- **`deploy.sh` の `.env` 非依存化**: ローカルは `.env` を source、CI は非機密の構成値（`GCP_PROJECT_ID` / `GCP_REGION` / `GCP_BUCKET_NAME`）を環境変数で渡す。機密値（API_KEY / GEMINI）は実行時に Secret Manager から注入するため、CI に機密を置かない。
- **`BACKEND_SHA` を Cloud Run サービス env に付与**: 「稼働中コードの版」の真実を backend SHA で持ち、冪等ガード/smoke を決定的にする。

## 検討した選択肢

### (A) デプロイ自動化の発火源

| 選択肢 | メリット | デメリット |
|--------|---------|----------|
| 手動 `deploy.sh` 継続（現状） | 仕組み不要 | 版ズレ事故が再発する。人手依存 |
| backend repo の push で発火 | backend に閉じる | 親リポの「リリースゲート」（意図的なポインタ更新）を無視し、未取り込みコミットまで本番化し得る |
| **親リポ main の gitlink 更新で発火** | ポインタ更新＝意図的リリースに一致。版ズレを根絶 | 親リポに CI 基盤と private submodule checkout の手当が必要 |

### (B) GCP 認証方式

| 選択肢 | メリット | デメリット |
|--------|---------|----------|
| SA 鍵 JSON を GitHub Secrets | 初期設定が簡単 | 長期秘密の保管・ローテ運用が必要。漏洩リスク（規約上 非推奨） |
| **Workload Identity Federation** | 長期鍵ゼロ・短命トークン・repo 限定で最小権限 | WIF pool/provider 等の初期構築が必要 |

### (C) 版ズレ検知の確実性

| 選択肢 | メリット | デメリット |
|--------|---------|----------|
| `paths` フィルタのみ | 設定が簡単 | force-push・多親マージ等で取りこぼし得る |
| **paths + 冪等 SHA ガード + smoke** | 取りこぼしても確実に検知・是正 | ステップがやや増える |

## 影響

- **新規**: `.github/workflows/deploy-backend.yml`（親リポ初の workflow）。
- **改修**: `infra/deploy.sh`（`.env` 非依存化、`BACKEND_SHA` 付与）、`infra/setup.sh`（`news-listen-sa` へ `cloudbuild.builds.editor` / `artifactregistry.writer` / `iam.serviceAccountUser` 追加、`SETUP_WIF=1` で WIF 構築）。
- **一回限りの前提セットアップ**:
  - GCP: `SETUP_WIF=1 bash infra/setup.sh`（WIF pool/provider と principalSet バインド）。
  - GitHub Secrets: `SUBMODULE_PAT`・`WIF_PROVIDER`・`WIF_SERVICE_ACCOUNT`。
  - GitHub Variables: `GCP_PROJECT_ID` / `GCP_REGION` / `GCP_BUCKET_NAME`。
- backend submodule・web・iOS には影響しない（親リポ内で完結）。
- **ロールバック**: smoke 失敗時は job が fail。`gcloud run services update-traffic news-listen-api --to-revisions=<前リビジョン>=100` で即時切戻し。イメージは `:latest` と `:<親sha>` が Artifact Registry に残るため `SKIP_BUILD=1` で任意版へ復帰可能。
- 既存の「ポインタを手動 PR で親 main に取り込む」運用は現状維持。本ワークフローはその merge を契機に自動発火する受け手。
