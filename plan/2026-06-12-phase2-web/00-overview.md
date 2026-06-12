# Phase 2 Web 残課題対応（統括）

**作成日:** 2026-06-12
**作業ブランチ:** `feature/web-tech-debt`（`feature/webui-restyle` から分岐。PR #7 未マージのためスタック構成）
**出典:** `docs/tech/web-frontend-tech-debt.md` #2・#3

| Task | 指示書 | 内容 | 占有ファイル |
|---|---|---|---|
| T01 | `01-test-type-errors.md` | テストの tsc 型エラー全 67 件をゼロ化（vi.mocked 化 + proxy の NextRequest 整合） | `web/tests/app/` 配下 7 テストファイルのみ |
| T02 | `02-seekbar-progress-fill.md` | シークバーの再生済み区間 amber フィル | `web/components/AudioPlayerBar.tsx`, `web/tests/components/AudioPlayerBar.test.tsx`, `web/app/globals.css`（T11 追加ブロックのみ） |

両タスクはファイル集合が互いに素であり**並列実行可**。

## 共通ルール

- coder は `.claude/agents/coder.md` 遵守。git 操作禁止
- テスト実行は担当ファイルに限定（`npx vitest run <パス>`）。全件検証はオーケストレーターが両タスク完了後に実施
- **既存テストの挙動アサーションを弱めない**（型注釈・キャストの追加は可、expect の削除・緩和は不可）
- 各指示書の「完了条件」は検証コマンドと期待出力つき。期待出力に到達しない試行が**同一ファイルで 3 回**続いたら停止してオーケストレーターに報告（coder.md の行き詰まりルール）

## 完了後の検証（オーケストレーター）

1. `cd web && npx tsc --noEmit 2>&1 | grep -c 'error TS'` → `0`
2. `npm run test` → 全件パス（315 + T02 追加分）
3. `npm run build` → 成功
4. git-composer がタスク単位コミット → push → PR（base: `feature/webui-restyle`）
5. `docs/tech/web-frontend-tech-debt.md` の #2・#3 を解消済みに更新
