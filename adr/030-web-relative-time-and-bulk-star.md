# ADR-030: Web 記事の相対日時表記（切替可）と複数選択一括スター

**ステータス:** 採用済み
**決定日:** 2026-06-25
**対象読者:** Web フロントエンド開発者
**関連:** [ADR-004](004-web-theme-dom-state.md)・[web-design.md](../design/web-design.md)・`web/lib/format.ts`・`web/contexts/AppContext.tsx`・PRD §13（P2）・issue #50

## 背景

PRD §13 の P2 課題に「相対日時表記（『3時間前』）」と「記事の一括スター」がある。記事カードの公開日は従来 `formatDate`（`M/D HH:MM` の絶対表記）固定で、過去の設計判断 D28 では相対表記化を一旦見送っていた。スター操作は 1 記事ずつのみで、まとめて star する手段が無かった。本 ADR で D28 の見送りを覆し、相対表記を**ユーザー選択式**で導入し、複数選択による一括スターを追加する。

## 検討した選択肢

| 論点 | 選択肢 | 評価 |
|---|---|---|
| 相対表記の適用 | (a) 絶対/相対をユーザー設定で切替（採用） | 既存利用者の絶対表記嗜好を壊さず包摂。localStorage 永続化のみで完結 |
| | (b) 相対表記へ一律置換 | D28 で見送った理由（絶対時刻を好む層）を再燃させる。却下 |
| 相対計算の現在時刻 | (a) `now` を引数注入（採用） | システム時刻非依存で決定的にテスト可能 |
| | (b) 関数内で `new Date()` 直接参照 | 時刻依存でテストが脆い。却下 |
| 一括スターの失敗扱い | (a) `Promise.allSettled` で部分成功を許容（採用） | 一部失敗でも成功分を反映し、成功/失敗件数を通知 |
| | (b) `Promise.all` で全件 or 全滅 | 1 件の失敗で全成功を捨てる。UX 劣化。却下 |

## 決定

- **相対日時**: `formatRelativeTime(date, now)` を `lib/format.ts` に純粋関数として追加（`たった今`/`N分前`/`N時間前`/`N日前`/`Nか月前`（30日単位）/`N年前`、未来は `もうすぐ`、不正値は空文字）。現在時刻 `now` は引数注入。
- **表示設定**: `AppState.timeFormat`（`'absolute' | 'relative'`、既定 `'absolute'`）を追加。`/settings` の「表示設定」セクションのセレクトで切替し、`setTimeFormat` が localStorage キー `time_format`（`lib/config.ts` の `KEY_TIME_FORMAT`）へ永続化。復元時に不正値は既定へフォールバック。`ArticleCard` は `timeFormat` に応じて `formatRelativeTime` か `formatDate` を選択。
- **一括スター**: `/feed` に選択モードを追加。ヘッダーの複数選択ボタンで開始、各 `ArticleCard` のチェックボックスで `selectedIds` をトグル、フッターの「N件を一括スター」で選択分を `Promise.allSettled(starArticle)` で一括登録。成功分のみ `starredIds` に反映し成功/失敗件数をトースト通知。実行中は対象 id を `busyIds` に入れて個別 Star/Dismiss と排他し、完了時に選択モードを解除。

## 理由

- 設定切替方式により D28 の懸念（絶対時刻を好む層）を壊さずに相対表記を提供できる。状態はクライアント完結で API 変更不要。
- `now` 引数注入で相対表記の境界（分/時/日/月/年・未来・不正値）を I/O なしで網羅テストできる。
- `Promise.allSettled` と busy 化により、ネットワーク部分失敗・個別操作との競合に対して堅牢に振る舞う。

## 結果（影響）

- 変更: `lib/format.ts`（`formatRelativeTime`）・`lib/config.ts`（`KEY_TIME_FORMAT`）・`contexts/AppContext.tsx`（`timeFormat`/`SET_TIME_FORMAT`/`setTimeFormat`）・`components/ArticleCard.tsx`（表記切替・選択チェックボックス）・`app/feed/page.tsx`（選択モード・一括スター）・`app/settings/page.tsx`（表示設定）。
- 受け入れ条件（テスト検証）: 相対表記の各境界、設定の永続化/復元/不正値フォールバック、一括スターの全成功・部分失敗・成功件数トースト・busy 排他。

## トレードオフと留保事項

- **月は 30 日固定・年は 365 日固定**の近似（暦月差ではない）。MVP として許容、表示の概数用途では十分。
- 相対表記は描画時点の `new Date()` を都度計算するため、長時間開いたままの画面では再描画されるまで更新されない（既知の軽微な制約）。
- 本 ADR は **Web のみ**。iOS の相対日時・一括スター（issue #50）は別途対応。
