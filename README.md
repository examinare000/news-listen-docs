# docs — ドキュメント索引

エージェント・人間が**必要な文書だけをオンデマンドで読む**ための索引（段階的開示）。
まず本索引で所在と「いつ読むか」を把握し、本体は必要時にのみ開く。

> **形式ポリシー**（`agent-rules/30-documentation-management.md`）: 仕様・設計の正本は **Markdown**（`docs/design/*.md`）。
> 人間向け HTML は MkDocs で生成（`mkdocs build`、`docs/design/*.html` の一部は生成物 / 一部は Phase 2b で Markdown 化予定）。

## 設計・仕様の正本（`docs/design/`）

| 文書 | 内容 | いつ読むか | 形式 |
|---|---|---|---|
| [backend-design.md](design/backend-design.md) | API 契約・データモデル・ジョブ・認証・キャッシュ・SSRF/レートリミット | バックエンドの仕様・挙動を知るとき | **Markdown（正本）** |
| [web-design.md](design/web-design.md) | API 契約・状態管理・画面/再生設計・受け入れ基準 | Web フロントの仕様を知るとき | **Markdown（正本）** |
| [index.html](design/index.html) | システム全体概要・技術スタック・コスト | 全体像をつかむとき | HTML（Phase 2b で Markdown 化予定） |
| [app-ui.html](design/app-ui.html) | Web UI の**視覚デザイン正本**（配色・フォント・レイアウト） | UI の見た目・トークンを知るとき | HTML（据え置き） |
| [ios-design.html](design/ios-design.html) | iOS アプリ設計（SwiftUI/MVVM） | iOS の設計を知るとき | HTML（Phase 2b で Markdown 化予定） |

## 意思決定・要件・計画・運用（Markdown）

| 区分 | 場所 | 内容 | いつ読むか |
|---|---|---|---|
| ADR | [adr/](adr/) | アーキテクチャ決定レコード（追記型） | 「なぜこの設計か」を知る・新決定を記録するとき |
| ADR（最新） | [adr/035-passkey-webauthn-adoption.md](adr/035-passkey-webauthn-adoption.md) | Passkey（WebAuthn/FIDO2）認証の採用 | 認証フロー・フィッシング耐性・セキュリティ設計を理解するとき |
| PRD | [prd/](prd/) | 要件定義・Phase 2 候補 | 要件・スコープを確認するとき |
| 計画 | [plan/](plan/) | 実装計画・タスク指示（作業領域・完了後削除） | 進行中タスクの計画を見るとき |
| 運用 | [operations/](operations/) | デプロイ状況・ローカル開発手順 | デプロイ・運用の現状を知るとき |
| 技術文書 | [tech/](tech/) | 技術的負債等 | 負債・技術メモを見るとき |
| 調査 | [research-reports/](research-reports/) | 調査・評価レポート（一覧） | 過去の調査結論を参照するとき |
| 調査（最新） | [research-reports/2026-06-26-antigravity-best-practices.md](research-reports/2026-06-26-antigravity-best-practices.md) | Antigravity 運用ベストプラクティスと適用プラン | エージェント（自身）の効率・精度向上プランを見るとき |

## 推奨読順

`README.md`（リポジトリ） → `docs/prd/` → `docs/design/`（正本） → `docs/adr/`
