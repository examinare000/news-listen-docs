# ADR-024: 日次ダイジェスト生成（type=digest ジョブ・キャッシュ外・決定論的 doc-id）

**ステータス:** 採用済み
**決定日:** 2026-06-25
**対象読者:** バックエンド開発者 / 運用担当者
**関連:** [ADR-006](006-cross-user-podcast-cache.md)（クロスユーザーキャッシュ／digest はその対象外）・[ADR-005](005-job-trigger-on-action.md)（ジョブ起動）・[ADR-023](023-tts-partial-failure-skip-join.md)（TTS 部分失敗）・`backend/jobs/digest_generator/main.py`・issue #43（PRD §5.2・§11 P1）

## 背景

`UserPrefs` に `digest_enabled` / `digest_article_count`（既定 5・範囲 3〜10）は実装済みだったが、ダイジェスト音声を生成するジョブが存在せず、generator は `type="single"`（1 記事 = 1 エピソード）のみだった。PRD §5.2/§11 P1 は「毎朝、Star＋高スコア記事を 1 本のダイジェストにまとめて配信する」を求める。

`Podcast` モデルは `type: Literal["single","digest"]` と `article_ids: list[str]` を既に持ち、複数記事を 1 エピソードに束ねる表現は可能だった。

## 検討した選択肢

| 選択肢 | メリット | デメリット |
|---|---|---|
| **専用ジョブ `jobs/digest_generator` を新設・キャッシュ外・決定論的 doc-id**（採用） | single 経路を汚さない。digest 固有の選定/冪等を隔離。既存 script/tts/storage を再利用 | ジョブが 1 つ増える |
| podcast-generator に digest 分岐を相乗り | ジョブ数を増やさない | single フローが複雑化。キャッシュ前提（ADR-006）と digest の非キャッシュ性が衝突 |
| digest もクロスユーザーキャッシュに載せる | 共有で再生成を減らせる | digest は各ユーザーの starred 集合＋当日スコアに依存し**共有不可**。前提が成立しない |

## 決定

- **専用ジョブ `jobs/digest_generator/main.py`** を新設する（Cloud Run Jobs。`USER_ID` env 必須）。既存の `ScriptGenerator`・`TtsGenerator`・`StorageClient`・`Notifier` を再利用する。
- **対象記事の選定**（純粋関数 `select_digest_articles`）: ユーザーの `starred_article_ids` ∩ 当日 `recommendation` のスコア付き記事を、score 降順の安定ソートで上位 `digest_article_count`（**3〜10 にクランプ**）件。対象 0 件（starred 無し／当日スコア無し）なら生成しない。
- **キャッシュ外（ADR-006 の対象外）**: digest はユーザー固有のため `PodcastCache`/`cache_key_for`/`try_acquire_cache` を一切使わず、合成音声を blob にアップロードして **`save_podcast`（type="digest", article_ids=複数, status="completed"）で直接保存**する。`CacheStatus` の 3 値は不変。
- **冪等性は決定論的 doc-id**: digest Podcast の id を `f"{user_id}_{YYYY-MM-DD}_digest"` とし、生成前に `get_podcast(id)` で存在チェック。存在すれば同日二重生成をスキップ（複合インデックス不要・O(1)・再実行で upsert）。
- **TTS 部分失敗（[ADR-023](023-tts-partial-failure-skip-join.md)）と整合**: 一部セグメント失敗時は成功分をスキップ結合し `status="partial_failed"` + `error_message` を記録。全失敗時は保存しない。
- **保存する難易度**は生成時の `prefs.default_difficulty`（固定値ではない）。
- **Scheduler**: Cloud Scheduler で毎朝（recommendation 後）起動する（`infra/deploy.sh`・親リポジトリ）。
- **gating**: `prefs.digest_enabled` が false のユーザーは生成しない。

## 理由

- digest を専用ジョブに隔離することで、キャッシュ前提の single 経路（ADR-006）を変更せずに済み、digest 固有の「選定・非キャッシュ・冪等」を 1 箇所に閉じ込められる。
- 決定論的 doc-id は、複合インデックスや範囲クエリを伴わずに「同日 1 本」を保証でき、ジョブ再実行（Scheduler のリトライや手動起動）に対して安全。
- digest はユーザーの starred 集合と当日スコアに依存し他ユーザーと共有できないため、ADR-006 のクロスユーザーキャッシュの前提が成立しない。キャッシュ外とするのが正しい。

## 結果（影響）

- 新ジョブ `jobs/digest_generator/`（`select_digest_articles`・`_build_digest_podcast`・`main`）。`ScriptGenerator` に `generate_digest(articles, difficulty, date)` と共有パーサ `_parse_script` を追加。
- digest Podcast は既存 `GET /podcasts` に `type="digest"` で現れるため、**クライアントは追加 API なしで一覧表示できる**（ダイジェスト専用 UI・ON/OFF/件数設定 UI はクライアント側の別 issue）。
- インフラ（`infra/deploy.sh`・`docker-compose.yml`・親リポジトリ）に digest-generator のジョブ定義と毎朝の Scheduler を追加。
- 受け入れ条件: 選定（starred∩scored・クランプ）・0 件スキップ・冪等・digest Podcast 保存・partial_failed をテストで検証（`tests/test_digest_generator_main.py`）。

## トレードオフと留保事項

- **難易度は単一**: digest は `prefs.default_difficulty` 一本で生成する。記事ごとに難易度を変える機能は持たない。
- **冪等の粒度は「日」**: 同日に starred が増えても、既に digest があれば再生成しない（翌日分で反映）。必要なら将来 doc-id にバージョンを足す。
- **クライアント UI**: ダイジェストの ON/OFF・件数（3〜10）設定 UI と専用表示は web/ios の別 issue。本 ADR は backend 生成と既存一覧での表示までを対象とする。
