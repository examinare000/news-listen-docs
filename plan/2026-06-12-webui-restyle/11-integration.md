# T11: 統合検証・シーム修正・Docker 確認

**Wave:** 3（T02〜T10 全完了 + 品質ゲート PASS 後に単独実行）
**依存:** T01〜T10
**占有ファイル:** 全ファイル編集可。ただし原則は `app/globals.css` の微調整と、各タスクの「残課題・申し送り」の解消のみ。コンポーネントの構造的変更が必要な場合は独断で行わずオーケストレーターに差し戻しを提案すること
**参照:** `00-overview.md` §5（受け入れ基準）、Wave 2 各 coder の完了報告（申し送り一覧はオーケストレーターから受領）

## 目的

並列実装された 10 タスクの成果を結合した状態で、デザイン再現度・機能リグレッション・Docker 動作を検証し、タスク間のシーム（申し送りされたスタイル不足・微ズレ）を解消する。

## 実装内容

### 1. 自動検証

- [x] `cd web && npm run test` — 全テストパス（315 passed / 22 files）
- [x] `npm run build` — エラーなし（`npm run lint` は ESLint 設定がリポジトリに存在せず対話セットアップを要求するため実行不能。設定追加はスコープ外として見送り、型検査は build が担保）
- [x] 禁止フォントの不存在確認: `grep -riE "Inter|Roboto|Arial|system-ui|Space Grotesk" web/app web/components` がフォント指定としてヒットしない（font-family は var(--font-*) のみ。next/font は Playfair Display / DM Sans / DM Mono）
- [x] 機密の不出力確認: `console.log` / `console.debug` が web/app・components・hooks・lib・contexts に存在しない

### 2. 申し送りの解消（globals.css 調整）

- [x] Wave 2 の各報告にある「不足スタイル」を globals.css 末尾にまとめて追加（既存セレクタは無変更）: `.toast-error` 配色 / `input[type=range]` のシーク・音量バー装飾（amber トラック + サム）/ `button.feed-tab` リセット / `.form-error` `.form-hint` `.form-success` / `.modal-actions` / `.content-narrow` / `.sr-only`。波形の一時停止状態は T06 実装のインライン animation-play-state で解消済みのため追加なし
- [x] 追加はデザイントークン（CSS 変数）を使い、app-ui.html の視覚言語から逸脱しないこと

### 3. ブラウザ実機確認（`npm run dev` + 実 API、または手順 4 の Docker 上）

00-overview.md §5 の手動シナリオ（T11 は CLI 環境のためブラウザ目視は実施不能。`npm run dev` + curl で全 5 ルートの SSR HTML に app-shell / sidebar / テーマ初期化スクリプト / aria-pressed タブ / role="switch" トグルが含まれることをスモーク確認済み。以下は**人間による確認推奨項目**として残す）:

- [ ] 初回フロー: localStorage クリア → SetupModal（新デザイン）→ 接続テスト → /feed
- [ ] テーマ: トグル即時切替・リロード維持・`prefers-color-scheme` 初期反映・ライトテーマでの可読性（コントラスト崩れがないか全 5 画面目視）
- [ ] フィード: タブフィルタ・件数・Star（★点灯 + トースト文言が D27 どおり）・Dismiss（カード除去）・更新
- [ ] Podcast: 一覧グリッド → 再生 → 再生中カード強調（amber 枠 + 波形）→ プレイヤーバーで ±シーク・速度・音量 → ページ遷移しても再生継続 → リロード後に位置復元
- [ ] 購読: 追加（409 文言）・削除（ConfirmDialog）・おすすめソース入力
- [ ] 設定: 保存・接続テスト・速度デフォルト反映
- [ ] レスポンシブ: 幅 900px 以下でサイドバーがアイコンのみ・375px で各画面が破綻しない（崩れは globals.css のメディアクエリ追記で対処し、内容を報告）

### 4. Docker 確認

```bash
docker compose up --build api web   # リポジトリルートで実行
```

- [x] ビルド成功（`docker compose build web` — Image news-listen-web Built。up は .env / SA 鍵依存のため未実施）
- [ ] http://localhost:3000 で新デザイン表示・SetupModal（バックエンド URL `http://api:8080` ではなく**ブラウザから到達可能な URL** — BFF プロキシはサーバー側転送のため `http://api:8080` を入力する。docker-compose.yml 冒頭コメントの手順に従う）※人間による確認推奨
- [ ] コンテナ上で手順 3 の主要シナリオ（フィード表示・再生・テーマ切替）を再確認 ※人間による確認推奨

### 5. 仕上げ

- [x] 不要になったデバッグコード・未使用クラス・未使用 import の除去（feed のインラインタブリセット定数を globals.css 化に伴い削除。console.log なし）
- [x] `docs/plan/2026-06-12-webui-restyle/` 各指示書のチェックボックスを実績に合わせて更新（Living Documentation）
- [x] 検証結果を coder.md の出力フォーマットで報告（このあと reviewer / ai-antipattern-reviewer の最終ゲート → git-composer が残コミット・PR 作成を行う）

## 完了条件

00-overview.md §5 の受け入れ基準 1〜5 を全て満たし、証跡（実行コマンドと結果）を報告に含めること。

## 禁止事項

- 新機能の追加（Phase 2 項目に触れない）
- 検証で発覚した設計レベルの問題の独断修正（オーケストレーターへ差し戻し提案）
- git 操作
