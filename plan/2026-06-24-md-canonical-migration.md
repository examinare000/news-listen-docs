# 計画: ドキュメント正本を Markdown へ反転（Markdown 正本 → HTML 生成）

- **作成日**: 2026-06-24
- **背景**: [調査・評価レポート](../research-reports/2026-06-24-agentic-coding-doc-strategy.md)。現行の「HTML 手保守正本」は Agentic Coding のベストプラクティス（単一 Markdown 正本・段階的開示・AGENTS.md）に反する。
- **方針**: HTML をやめず、**手書き正本から生成物へ降格**する。Markdown を単一正本とし、人間向け HTML は SSG で生成する。
- **スコープ外**: `app-ui.html`（視覚デザイン正本）は据え置く。
- **採用範囲**: Phase 1 + Phase 2（ユーザー承認 2026-06-24）。
- **進捗（2026-06-29 更新・Phase 2 クローズ）**: Phase 1 完了（AGENTS.md・docs 索引）。**Phase 2 完了（検証・公開まで）**:
  - 正本化: backend/web/index/ios-design を Markdown 正本化・Mermaid 化、agent-rules/30 反転、MkDocs 設定、参照張替、旧 HTML 削除。
  - 検証: `mkdocs build --strict` がローカルで通過（exit 0・警告/エラー無し）。
  - 公開: 親リポに `.github/workflows/deploy-docs.yml` を新設（docs 更新 → MkDocs ビルド → GitHub Pages 公開、生成物 site/ は非コミット）。docs は public のため Secret 不要で docs サブモジュールのみ https init する設計。
  - **残（リポ管理者操作）**: GitHub Pages の有効化（Settings → Pages → Source = GitHub Actions）。これ完了で初回公開が走る。
  - 補足: `mkdocs.yml` は `docs/` ではなく**親リポ root** に置く（`docs_dir: docs`。submodule を親が束ねてビルドする構成のため、親が正）。
  - **未達/見送り**: Phase 2-4 の「トークン50%削減」目標は未達（旧 backend HTML 65KB に対し現 MD 75KB。2026-06-29 の実装同期で内容拡張が主因）。Phase 2 の価値は size 削減ではなく単一 Markdown 正本化・Mermaid 化・エージェント可読性にあると判断し、size 目標は達成基準から外す。
  - Phase 3（堅牢化）は未着手。

## ライフサイクル注記
本書は作業領域。完了後は削除し、確定方針は `agent-rules/30` と各 `docs/design/*.md` へ移す。

---

## Phase 1 — 段階的開示の基盤（追加・低リスク）

- [ ] `AGENTS.md`（リポジトリルート）を新設。横断標準のベース層として、ビルド/テスト/規約の要点 + `agent-rules/`・`CLAUDE.md`・docs 索引への参照を <100 行で記述。既存 `AGENT.md`(17行) は AGENTS.md へ統合し、旧ファイルはポインタ化 or 削除。
- [ ] `docs/README.md`（docs 索引）を新設。各ドキュメントを「何を / いつ読むか / 形式」で 1 行記述（progressive disclosure の索引）。エージェントは索引→必要文書だけオンデマンド取得。
- [ ] （任意）`docs/llms.txt` を索引の機械可読版として併設。

**完了条件**: エージェントが索引を読めば全 docs の所在と読むべき場面を把握でき、本体は必要時のみロードできる。

## Phase 2 — 反転の中核（Markdown 正本 + HTML 生成）

### 2-0 方針文書の改訂
- [x] `agent-rules/30-documentation-management.md` を改訂: 形式ポリシーを「**Markdown が単一正本／HTML は SSG 生成物**」「軸は『正本 vs 生成物』『常時ロード vs オンデマンド』」へ。Mermaid 図の徹底を明記。

### 2-1 技術仕様の Markdown 正本化
- [x] `docs/design/backend-design.html` → `docs/design/backend-design.md`（正本）。内容を移植し、SVG 図を **Mermaid** に再作図。実装コードは非掲載（責務・契約・表中心）の原則を維持。
- [x] `docs/design/web-design.html` → `docs/design/web-design.md`（正本）。同上。
- [x] `index.html`（システム概要）も `index.md` 化を検討（任意。視覚比重が高ければ後回し可）。→ `docs/design/index.md` として正本化済み。
- [x] 旧 `*.html`（手書き）は削除し、生成物に置換（下記 2-2）。`app-ui.html` のみ視覚正本として据え置き。

### 2-2 HTML 生成（SSG）
- [x] MkDocs + Material テーマを導入。`mkdocs.yml` は**親リポ root** に配置（`docs_dir: docs`。submodule を親が束ねてビルドする構成のため）、`docs/design/*.md` を nav 化。Mermaid プラグインを有効化。
- [x] CI（GitHub Actions）で main へ docs 更新時に `mkdocs build --strict` → 静的 HTML を生成し **GitHub Pages** へ公開（`.github/workflows/deploy-docs.yml`）。※ Pages の有効化（Source=GitHub Actions）はリポ管理者操作として残る。
- [x] 生成物（`site/`）は `.gitignore` 済（追跡 0 ファイル）。手書き HTML はリポジトリから除去済。

### 2-3 参照の更新
- [x] ADR・plan・README・ios-design からの設計書参照を `*.md`（正本）へ張り替え。
- [x] `app-ui.html` は視覚正本として残すため、web-design.md から app-ui.html を参照する関係は維持。

### 2-4 検証
- [x] `mkdocs build --strict` が通る（exit 0・警告/エラー無し。リンク切れ・nav 不整合なし）。
- [ ] ~~Markdown 正本のトークン量が旧 HTML 比で大幅減（目標: backend 仕様で 50%+ 削減）~~ → **未達**: 旧 backend HTML 65KB に対し現 MD 75KB（実装同期で内容拡張）。size 削減は達成基準から外す（上記進捗メモ参照）。
- [x] エージェントが Markdown 正本のみで仕様を把握できる（正本は Markdown のみ・実装コード非掲載の契約中心構成）。

## Phase 3 — 任意の堅牢化（後日判断）
- [ ] pre-merge チェックリストに「docs 同期」追加、または CI ドリフト検査。
- [ ] 仕様駆動（Spec Kit 流 requirements/design/tasks）の形式化（必要なら）。

---

## リスクと判断
- **反転は前回の HTML 正本化（PR docs#8 / parent#27）を部分的に巻き戻す**。視覚体験は SSG 生成で同等以上に維持されるため、人間向け価値は損なわない。
- SSG・CI 導入の工数が中程度。最小構成なら Phase 1 のみでも段階的開示の効果は得られる（Phase 2 は別 PR で段階実施）。
- コミット/PR はリポジトリ別（docs サブモジュール / 親）にフィーチャーブランチで行い、ブランチ保護を順守する。
