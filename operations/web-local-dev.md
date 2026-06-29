# Web ローカル開発・検証手順

**最終更新:** 2026-06-29（ADR-037 / env 手順追記）

## リポジトリ構成
本ドキュメントが含まれる `docs/` および `web/` は、現在は独立したリポジトリ（Git Submodule）として管理されています。
開発の際は、親リポジトリから `web/` ディレクトリに移動して作業を行います。

## 日常の開発コマンド
| 目的 | コマンド | 備考 |
|---|---|---|
| dev（開発サーバー起動） | `npm run dev` | http://localhost:3000 で起動。ホットリロード対応 |
| build | `npm run build` | Next.js production ビルド。Vercel デプロイ時と同等 |
| test | `npm run test` | Vitest で全テスト実行 |
| lint | `npm run lint` | ESLint (Flat Config)・構文・スタイルチェック |
| 型チェック | `npm run type-check` | TypeScript 型安全性検証 |

## ADR-037: ローカル環境変数設定

ローカルで backend との通信をテストする際は、`.env.local` ファイルを `web/` ディレクトリに作成し以下を設定（[ADR-037](../adr/037-gateway-api-key-client-distribution.md) 参照）:

```env
BACKEND_BASE_URL=https://news-listen-api-ck5vowuina-an.a.run.app
BACKEND_API_KEY=<本番 backend の API_KEY>
```

または、Docker Compose で backend をローカル起動している場合:

```env
BACKEND_BASE_URL=http://api:8080
BACKEND_API_KEY=<ローカル API キー>
```

`npm run dev` 実行時、Next.js BFF サーバーが backend 呼び出しに使用します。
