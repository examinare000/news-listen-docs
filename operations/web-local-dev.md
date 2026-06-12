# Web ローカル開発・検証手順

**最終更新:** 2026-06-12
**対象読者:** Web フロントエンド開発者

## 日常の開発コマンド（`web/` で実行）

| 目的 | コマンド | 備考 |
|---|---|---|
| 開発サーバー | `npm run dev` | http://localhost:3000 |
| 全テスト | `npm run test` | Vitest。10 秒以内に完走 |
| ビルド検証 | `npm run build` | 型検査を含む。`tsconfig.json` / `next-env.d.ts` が自動更新されることがある（コミット対象。ADR-003 参照） |
| lint | 使用不可 | ESLint 未導入（`docs/tech/web-frontend-tech-debt.md` #1）。**`npx next lint` を実行しないこと** — 対話セットアップに入り `tsconfig.json` を自動改変する |

## Docker での動作確認

```bash
# リポジトリルートで実行
docker compose up -d --build api web
```

- **`docker compose build` だけではコンテナは入れ替わらない。** コード変更を反映するときは必ず `up -d --build` まで実行する（build のみで「スタイルが当たらない」等、古いイメージのコンテナを見続ける事故の前例あり）
- api コンテナは `.env` と SA 鍵（`infra/keys/news-listen-sa.json`）が必要。web 単体の再作成には不要
- SetupModal の入力値: バックエンド URL = `http://api:8080`（BFF プロキシがサーバー側転送するためコンテナ間名でよい。ADR-001）、API キー = `.env` の `API_KEY`

## リリース前の手動確認シナリオ

仕様書 `docs/spec/2026-06-10-web-frontend-spec.md` §14（受け入れ基準）を正とし、UI 変更時は特に以下をブラウザで目視する:

1. 初回フロー: localStorage クリア → SetupModal → 接続テスト → /feed
2. テーマ: トグル即時切替・リロード維持・`prefers-color-scheme` 初期反映・ライトテーマ全画面のコントラスト
3. フィード: タブフィルタ・Star（★点灯 + トースト文言）・Dismiss・更新
4. Podcast: 再生 → 再生中カード強調 → プレイヤーで ±シーク・速度・音量 → ページ遷移しても再生継続 → リロード後の位置復元
5. 購読: 追加（重複 409 文言）・削除（確認ダイアログ）・おすすめソース入力
6. レスポンシブ: 幅 900px 以下でサイドバーがアイコンのみ・375px で破綻なし
