# ADR-028: レコメンド履歴分析への Gemini Context Caching 導入（安全縮退付き）

**ステータス:** 採用済み
**決定日:** 2026-06-25
**対象読者:** バックエンド開発者 / 運用担当者
**関連:** [ADR-005](005-job-trigger-on-action.md)（Star/Dismiss でレコメンドジョブ再トリガ）・[ADR-006](006-cross-user-podcast-cache.md)・`backend/shared/gemini_client.py`・`backend/jobs/recommendation/recommender.py`・PRD §10・§7・issue #52

## 背景

PRD §10/§7 は「Context Caching で日次履歴分析コストを削減」を前提としていたが、レコメンドの Gemini 呼び出しは Context Caching 未使用だった。レコメンドのプロンプトは**安定部分（指示＋ユーザー履歴 starred/dismissed）**と**可変部分（候補記事）**に分かれ、安定部分は同一ユーザー・同一履歴で繰り返し送られている。

重要な前提として、[ADR-005](005-job-trigger-on-action.md) により **Star/Dismiss がレコメンドジョブを再トリガ**する。活動的なユーザーは**同日内にレコメンドが複数回**走り、毎回ほぼ同じ大きな履歴コンテキストを再送している。

## 検討した選択肢

| 選択肢 | メリット | デメリット |
|---|---|---|
| **安定コンテキストを explicit Context Caching・効かない時は通常呼び出しへ安全縮退**（採用） | 同日再実行で履歴トークンを再課金せず削減。結果は不変 | キャッシュ作成オーバーヘッド。1 回限りや小履歴では効果が出ない |
| 何もしない（現状維持） | 単純 | PRD のコスト前提を満たさない |
| 候補をバッチ分割して多数回呼ぶ | キャッシュ再利用回数が増える | スコアの全体正規化が崩れる挙動変更。過剰 |

## 決定

- `GeminiClient` に追加:
  - `generate_text_with_usage(prompt, *, cached_content=None, temperature)` → `TextGenerationResult(text, prompt_token_count, cached_content_token_count)`。`usage_metadata` 欠落時は 0 に安全化。既存 `generate_text(...) -> str` は**非破壊で据え置き**。
  - `create_cached_content(system_instruction, *, display_name, ttl_seconds) -> str | None`（`caches.create`。**作成失敗・min-token 未満時は None**＝縮退合図、例外は warning で握る）。
  - `find_cached_content(display_name) -> str | None`（`caches.list` を走査し display_name 一致を返す。ジョブは都度プロセス起動のため**内部マップを使わずクロスプロセス再利用**する）。
  - 純粋関数 `recommendation_cache_display_name(user_id, stable_context) -> str`（履歴の SHA-256 で決定的に命名）。
- `recommender.score_articles`: 安定コンテキスト（指示＋履歴）を display_name で **find-or-create** し、可変部分（候補）で `generate_text_with_usage(cached_content=...)`。**キャッシュ名が得られない場合（min-token 未満／失敗）は従来の完全プロンプトで通常呼び出しへ縮退**する。スコアリングのパース・結果は**キャッシュ有無で不変**。
- **安全縮退の不変条件**: キャッシュが効かない場合、通常呼び出しと**同一結果・同一コスト**に縮退する（出力フォーマット指示は可変側プロンプトに含まれ、キャッシュ経路でも JSON 出力は保たれる）。
- **計測**: `usage_metadata.cached_content_token_count` をログに記録し削減を可視化する。TTL は数時間（6h）。履歴が変われば display_name も変わり新規キャッシュ・旧キャッシュは TTL で失効。

## 理由

- 安定部分を分離してキャッシュし、可変部分のみ毎回送ることで、ADR-005 由来の同日再実行で履歴トークンの再課金を避けられる。
- 「効かない時は通常呼び出しに完全縮退」を不変条件にすることで、min-token 未満・SDK 障害・1 回限りのケースでも**結果とコストが悪化しない**安全性を担保する。
- `cached_content_token_count` のログにより、受け入れ条件「削減（計測）」を観測可能にする。

## 結果（影響）

- `shared/gemini_client.py`・`jobs/recommendation/recommender.py` を改修（main の挙動・スコア結果は不変）。
- 受け入れ条件: キャッシュヒット時に `cached_content_token_count > 0` をテストで pin。フォールバック経路でスコアが不変であることをテストで固定。Gemini SDK は全面モックで検証（実 API 非呼び出し）。

## トレードオフと留保事項（効果の正直な評価）

- **削減が出る条件**: 「同日に複数回レコメンドが走る（[ADR-005](005-job-trigger-on-action.md) のトリガ）」かつ「履歴が explicit cache の **min-token 下限以上**」。この条件下で 2 回目以降に履歴トークンが割引課金される。
- **効果が出ない/わずかに増えるケース**: 履歴が小さい（min-token 未満）・1 日 1 回しか走らない。これらでは縮退して通常呼び出しと同等コストに留まる（悪化しない設計）。
- **`caches.list` のコスト**: レコメンドごとに 1 回のキャッシュ一覧取得が増える（再利用判定のため）。低頻度ジョブのため許容。
- 自動的なキャッシュ削除や複数ユーザーをまたぐバッチ最適化は本 ADR の対象外。
