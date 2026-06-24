# Agentic Coding におけるドキュメント戦略の調査と現行戦略の批評的評価

- **対象読者**: 本プロジェクトの設計・運用担当（人間 / エージェント）
- **作成日**: 2026-06-24
- **目的**: Agentic Coding のドキュメント戦略ベストプラクティスを調査し、現行戦略（HTML 正本 / Markdown 作業層）の妥当性を批評的に評価。より洗練された手法の採否を判断する。
- **結論（先出し）**: 現行戦略の中核判断「**HTML を手保守の仕様正本にする**」は業界コンセンサスに反する。**「Markdown を単一正本とし HTML は生成する」への反転**を推奨する。

---

## 1. 調査方法

Web 調査（3 観点の並列調査）で一次情報・標準・ベンダ公式・著名エンジニアリングブログを収集。観点は (A) AI 向けドキュメント形式・標準、(B) 仕様駆動開発・Living Doc、(C) Anthropic の context engineering。出典は §6。

## 2. 業界コンセンサス（要点）

| 論点 | コンセンサス | 根拠 |
|---|---|---|
| AI が読む正本の形式 | **Markdown が最適**。HTML→MD でトークン 20〜67% 削減、recall/精度向上。タグ・CSS・SVG はノイズ | llmstxt.org、各種ベンチ、Anthropic「context rot」 |
| 正本と表示の関係 | **単一の Markdown 正本を保持し HTML は生成**（docs-as-code / SSG）。HTML 手保守正本は非推奨＝ドリフト源 | docs-as-code、MkDocs / Docusaurus |
| エージェント指示の標準 | **AGENTS.md**（Markdown・横断標準）。存在時に実行時間 約 28.6% 短縮・出力トークン 16.6% 減の報告 | agents.md、arXiv |
| 段階的開示 | 索引先読み→必要時に本体取得（llms.txt、Claude Skills 方式）。常時ロードは小さく | Anthropic context engineering、Kiro / Cursor |
| context engineering | 「最小の高シグナルなトークン集合」を維持。肥大は attention budget を食い recall 低下 | anthropic.com/engineering 公式 |
| 仕様駆動 | 仕様/設計/タスクは Markdown で段階生成（Spec Kit、AWS Kiro）。ADR は追記型 Markdown | github/spec-kit、kiro.dev |

## 3. 現行戦略の批評的評価

現状: **HTML（`docs/design/*.html`）＝設計・仕様の正本**、Markdown＝ADR/計画/運用。判断軸は「更新頻度」。

### 妥当な点
- ADR/PRD/計画/運用を Markdown に置くのはコンセンサスと一致。
- 「HTML は人間の理解を助ける」直感は正しい — ただし*生成された表示*の利点であって*手保守の正本*の利点ではない。
- CLAUDE.md（全て 70 行未満）+ agent-rules の階層化は Anthropic 指針（<200 行・段階的）に合致。指示レイヤーは健全。

### 問題点（証拠付き）
1. **分割軸の取り違え**: 正しい軸は「正本 vs 生成物」「常時ロード vs オンデマンド」。更新頻度で割った結果、最も AI 価値の高い仕様を最も AI に不利な形式に置いた。
2. **HTML 正本は実測で AI に不利**: `backend-design.html` は 738 行 / 65KB / SVG 要素 97 個（≈16K トークン、多くが CSS/SVG ノイズ）。`app-ui.html` は CSS だけで 1351 行。読むたびに attention budget を消費し context rot で recall 低下。
3. **掲げた目的の自己矛盾**:
   - 「二重管理・乖離防止」のため HTML 化 → docs-as-code の結論は逆（MD 正本→HTML 生成）。手書き HTML 正本はアンチパターン。
   - 「Markdown＝AI がコードベース全調査せず把握する圧縮層」と定義 → コードベースを圧縮した当の仕様を AI 非互換の HTML に置いた。論理が反転。
4. **自前ルールとの緊張**: agent-rules/30 は「図は Mermaid 等」を規定。手書き SVG は人間には描画されるが AI/差分には不透明。

## 4. より洗練されたモデル（両目的を両立）

**正本と表示を反転**:
- 単一の **Markdown 正本**（Mermaid 図入り）— AI・git 差分に最適、単一ソースでドリフト消滅。
- **HTML は CI で生成**（MkDocs Material / Docusaurus）— 人間の体裁は維持、手保守ゼロ。
- **索引先読み**（`AGENTS.md` + docs 索引 / `llms.txt`）— 索引→必要な仕様だけオンデマンド取得。
- **AGENTS.md 採用**（現状は標準形でない `AGENT.md` 17 行のみ）— 横断標準のベース層に統一。

「HTML をやめる」のではなく「**HTML を手書き正本から生成物へ降格**」する案。人間体験を保ったまま AI 可読性・単一正本・差分追跡を回復する。

## 5. 採用方針

[2026-06-24-md-canonical-migration.md](../plan/2026-06-24-md-canonical-migration.md) のとおり Phase 1+2 を採用（反転）。app-ui.html（視覚デザイン正本）は据え置き可。

## 6. 出典

- https://llmstxt.org/ ・ https://agents.md/ ・ https://www.infoq.com/news/2025/08/agents-md/
- https://web2md.org/blog/markdown-vs-html-for-llm ・ https://www.searchcans.com/blog/html-vs-markdown-llm-context-window-optimization/
- https://konghq.com/blog/learning-center/what-is-docs-as-code ・ https://github.com/mkdocs/mkdocs
- https://github.github.com/spec-kit/ ・ https://kiro.dev/docs/steering/ ・ https://martinfowler.com/articles/exploring-gen-ai/sdd-3-tools.html
- https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents
- https://code.claude.com/docs/en/memory ・ https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills
- arXiv（AGENTS.md 効率）: https://arxiv.org/pdf/2601.20404 ・ https://arxiv.org/pdf/2602.11988
