# ADR-029: 記事全文検索（Firestore 制約下のアプリ層フィルタ MVP）

**ステータス:** 採用済み
**決定日:** 2026-06-25
**対象読者:** バックエンド開発者
**関連:** [ADR-006](006-cross-user-podcast-cache.md)・`backend/api/routers/articles.py`・PRD §5.1・§13・issue #51

## 背景

取得済み記事のキーワード全文検索が無く（PRD §5.1/§13 P2）、フィードはレコメンド順のみだった。**Firestore はネイティブ全文検索を持たない**（前方一致・`array_contains`・等号/範囲のみ）。最小実装で「キーワードで取得済み記事を検索できる」を満たす。

## 検討した選択肢

| 選択肢 | 実装複雑度 | スケール適性 | 評価 |
|---|---|---|---|
| **(a) アプリ層フィルタ（最新200件をメモリで部分一致）**（採用） | 低 | 記事 200〜500 件で実用 | **MVP に最適** |
| (b) `keywords` トークン配列 + `array_contains` 索引 | 高 | 大量記事も可 | 日本語形態素解析・索引メンテが重い。YAGNI |
| (c) 外部検索（Algolia 等） | 中 | 無限 | 追加 SaaS・コスト・外部依存。過剰 |

## 決定

- **`GET /articles/search`**（要認証）を追加する。クエリ: `q`（必須・`min_length=1`・`max_length=100`）・`limit`（既定20・1〜200）・`offset`（既定0）・`filter`（`all`/`unread`）。
- 検索ロジックは**純粋関数 `_search_articles(articles, query, read_ids, dismissed_ids, filter)`** に切り出す（I/O なし・テスト容易）。
  - `db.get_recent_articles(limit=200)` の最新記事を対象に、`q` を `strip().lower()` して **title または content への部分一致**（Python の `in`＝UTF-8 substring・大文字小文字不問・日本語対応）。
  - **空白のみクエリは `strip` 後 `""` になり `"" in s` が常に真で全件マッチしてしまうため、空クエリは結果なしとして早期 return**（`Query(min_length=1)` は空白を弾けない）。
  - `filter="all"` は dismissed のみ除外、`filter="unread"` は read ∪ dismissed を除外。入力順（published_at DESC）を保持。
- レスポンスは新スキーマ `ArticleSearchResponse{ articles: list[ArticleResponse]; total_count: int }`。`total_count` は**ページング前の全一致件数**、`articles` は `offset/limit` でスライス。各 `ArticleResponse.is_read` は `a.id ∈ read_ids`。read-only のため**監査記録なし・ジョブ起動なし**。

## 理由

- 記事 200〜500 件規模・利用者 1〜5 人の現状では、メモリ上の部分一致が最小実装かつ十分高速（< 10ms 想定）で、Firestore 索引追加も Article モデル変更も不要。
- 検索可否ロジックを純粋関数に隔離し、部分一致・大文字小文字・日本語・除外・空クエリ・順序を I/O なしで網羅テストできる。
- 空白クエリの早期 return により「`"" in s` で全件返る」事故を構造的に防ぐ。

## 結果（影響）

- `api/routers/articles.py` に `_search_articles` と `GET /articles/search`、`api/schemas.py` に `ArticleSearchResponse`。Firestore メソッドの追加は不要（`get_recent_articles` 流用）。
- 受け入れ条件: title/content 部分一致・大文字小文字不問・日本語・dismissed/read 除外・**空白クエリで空**・ページング（total_count はページング前）・is_read・401/422 をテストで検証。

## トレードオフと留保事項

- **検索範囲は最新 200 件**: それより古い記事は対象外（MVP の既知制約）。記事数が数千件規模に増えたら (b) keywords 索引や (c) 外部検索へ段階的に移行する。
- **スコアは中立 0.5 固定**: 検索結果はレコメンドスコアを反映しない（関連度順ではなく published_at 順）。関連度ランキングは将来課題。
- **クライアント UI（web/ios の検索画面）は別 issue**。
