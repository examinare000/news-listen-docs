# Web フロントエンド技術的残課題

**最終更新:** 2026-06-12（Phase 2 第1弾でモック型エラー・シークバーフィルを解消し削除）
**対象読者:** Web フロントエンド開発者
**出典:** WebUI リスタイル（PR #7）完了時点の棚卸し

機能的な Phase 2 候補は PRD（`docs/prd/2026-05-31-news-listen.md` §11）を参照。本書はコード品質・開発基盤の残課題のみを扱う。解消済み項目は本書から削除する（経緯は git 履歴）。

| # | 課題 | 内容 | 対応の目安 |
|---|---|---|---|
| 1 | ESLint 未導入 | `web/` に ESLint 設定が存在せず `npm run lint` が対話セットアップで停止する。型検査は `npm run build`（tsc）が担保している | flat config（eslint.config.mjs）+ next/core-web-vitals を導入する独立タスク。**`npx next lint` を場当たりに実行しないこと**（tsconfig.json を自動改変する） |
| 2 | `.feed-tabs` の背景色リテラル | `rgba(11,10,14,.88)` がダーク固定でライトテーマ時に不正確（デザイン正本 `app-ui.html` 由来の値をそのまま移植）。`--bg-glass-header` 変数への置換が候補だが、既存セレクタ無変更の原則（ADR-003）によりデザイン正本の更新とセットで行う | デザイン正本の修正 → globals.css 同期 |
| 3 | Toast error の `role` | success/error とも `role="status"` 系。spec §10.6 は error に `role="alert"` を想定 | 実装と spec の突き合わせ確認の上、どちらかに寄せる |

## 解消済み（2026-06-12・feature/web-tech-debt）

- **テストのモック型エラー 67 件 → 0 件**: `vi.mocked()` + `as unknown as ReturnType<typeof createApiClient>` キャストへ統一（proxy.test.ts の `NextRequest` 型は `asNextRequest` キャストヘルパー）。以後、新規テストもこのパターンに従うこと（`npx tsc --noEmit` のエラー 0 を維持）
- **シークバーの再生済み区間フィル**: `--seek-fill` カスタムプロパティをインライン付与（ゼロ除算ガード + clamp）し、WebKit は runnable-track の `linear-gradient`、Firefox は `::-moz-range-progress` で描画
