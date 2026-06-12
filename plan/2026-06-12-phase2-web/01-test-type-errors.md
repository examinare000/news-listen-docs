# T01: テストの tsc 型エラー全 67 件のゼロ化

**占有ファイル（これ以外は一切編集禁止。実装コード・`lib/api.ts`・`vitest.config.ts`・`tsconfig.json` も編集禁止）:**
- `web/tests/app/subscriptions/page.test.tsx`（21 件）
- `web/tests/app/api/proxy.test.ts`（15 件）
- `web/tests/app/feed/page.test.tsx`（11 件）
- `web/tests/app/podcast/id/page.test.tsx`（9 件）
- `web/tests/app/podcast/page.test.tsx`（8 件）
- `web/tests/app/settings/page.test.tsx`（2 件）
- `web/tests/app/page.test.tsx`（1 件）

**参照（読むだけ）:** `web/tests/components/ui/SetupModal.test.tsx` L120〜135 — vi.mocked 化の確立済み先例

## 背景（2026-06-12 時点の実測。着手前に再計測して一致を確認すること）

`cd web && npx tsc --noEmit` で **67 件**。内訳:
- **TS2339 × 50**: `createApiClient.mockReturnValue` — `vi.mock('@/lib/api', ...)` でモック済みだが、import した `createApiClient` の静的型がモック型でないため
- **TS2345 × 15**: `proxy.test.ts` のみ — Web 標準 `Request` を `NextRequest` 引数に渡している（`cookies, nextUrl, page, ua` 不足）
- **TS2554 × 2**: `Expected 1 arguments, but got 0` — 該当 2 行を実際に読み、引数なし呼び出しの実態に合わせて最小修正する

## 修正レシピ（この 3 パターン以外の修正方法を発明しないこと）

### パターン A: TS2339（mockReturnValue）→ vi.mocked 化

各 `createApiClient.mockReturnValue({...})` を先例（SetupModal.test.tsx）と同形に置換:

```
vi.mocked(createApiClient).mockReturnValue({...従来と同じオブジェクト...} as unknown as ReturnType<typeof createApiClient>)
```

- モックオブジェクトの**中身は 1 文字も変えない**（型キャストで包むだけ）
- ファイル内で同じ置換が多数あるため、置換後に `grep -n 'mockReturnValue' <ファイル>` で全箇所が `vi.mocked(` 経由になったことを機械的に確認する
- `mockResolvedValue` / `mockRejectedValue` 等の同型エラーがあれば同じレシピを適用

### パターン B: TS2345（proxy.test.ts の NextRequest）

ファイル先頭付近にヘルパーを 1 つ定義し、エラー行の `Request` 値を包む:

```
const asNextRequest = (req: Request) => req as unknown as NextRequest
```

- `NextRequest` の import が必要なら `import type { NextRequest } from 'next/server'` を追加
- 実際に `new NextRequest(...)` へ作り替える方式は**採らない**（ランタイム挙動が変わるリスクがあるため。型キャストのみ）

### パターン C: TS2554（2 件）

該当行を読み、呼び出し先のシグネチャに合わせて**テスト側に**最小の引数を追加する（呼び出し先の実装は変更しない）。判断に迷う場合はその行の前後のテスト意図を読み、既存テストが期待する挙動を変えない引数を選ぶ。

## 手順（TDD ではなく型レベルのリファクタリング。挙動は不変）

1. 着手前の基線計測: `cd web && npx tsc --noEmit 2>&1 | grep -c 'error TS'` と、担当 7 ファイルの vitest がグリーンであることを確認: `npx vitest run tests/app/`
2. **1 ファイルずつ**修正 → そのファイルの vitest 実行 → tsc 残件数が単調減少していることを確認、の順で進める（一括 sed での全ファイル同時置換は禁止。1 ファイル内での sed/Edit の使用は可）
3. 全ファイル完了後、完了条件の 3 コマンドを実行

## 完了条件（期待出力つき）

| コマンド | 期待出力 |
|---|---|
| `cd web && npx tsc --noEmit 2>&1 | grep -c 'error TS'` | `0` |
| `cd web && npx vitest run tests/app/` | 担当分全テストパス（fail 0。着手前と同数以上の pass） |
| `cd web && grep -rn 'createApiClient.mockReturnValue' tests/ | grep -v 'vi.mocked' | wc -l` | `0` |

## 停止条件（処理ループ防止）

- 同一ファイルへの修正を 3 回試みても tsc エラーが減らない → そのファイルを未着手状態（直前のグリーン状態）に戻して停止し、残エラーの全文を添えて報告
- レシピ A〜C に当てはまらないエラーに遭遇した → 修正せず、エラー全文を添えて報告（独自レシピの発明禁止）
- `npx tsc --noEmit` の総実行回数が 25 回を超えた → 停止して進捗と残件を報告

## 禁止事項

- expect / アサーションの削除・緩和・skip 化（型の問題を挙動テストの弱体化で解消しない）
- 実装コード（`web/app/`・`web/components/`・`web/lib/`）と設定ファイルの変更
- `any` 型注釈の散布（キャストは `as unknown as <具体型>` の形に限定）
- 占有外テストファイル（`tests/components/` 等）への変更
