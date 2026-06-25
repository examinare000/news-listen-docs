# ADR-006: クロスユーザー Podcast キャッシュと待機者調整の方式

**ステータス:** 採用済み
**決定日:** 2026-06-13
**対象読者:** バックエンド開発者
**関連:** ADR-005（Star/Dismiss を契機にジョブを自動起動する）・[ADR-024](024-daily-digest-generation.md)（日次ダイジェストは本キャッシュの**対象外**）・[ADR-025](025-storage-management-safe-cleanup.md)（一括削除は共有キャッシュ blob を**削除しない**）

> **補足（2026-06-25・issue #43）:** 本 ADR のクロスユーザーキャッシュは `type="single"` を対象とする。日次ダイジェスト（`type="digest"`）はユーザーの Star 集合＋当日スコアに依存し共有できないため、[ADR-024](024-daily-digest-generation.md) でキャッシュ対象外とし、決定論的 doc-id で per-user に直接保存する。

## 背景

Podcast 生成（`backend/jobs/podcast_generator/main.py`）の重複排除は現状ユーザー単位でしか働かない。`FirestoreClient.podcast_exists_for_article(user_id, article_id, difficulty)` は `user_id` を含むクエリであり、音声 blob も `podcasts/{podcast_id}/{difficulty}.mp3`（`podcast_id` は UUID）とユーザーごとに別物になる。

ADR-005 によりスター操作は **per-user の** `podcast-generator` ジョブを起動する。debounce ロックは `jobLocks/{user_id}_{job_name}` と **ユーザー別キー** なので、複数ユーザーが同じ記事をスターしても互いをブロックせず、それぞれのジョブが走る。結果として `(article_id, difficulty, language)` が一致しても **Gemini スクリプト生成 + TTS が人数分** 走り、最大のコスト/レイテンシ要因（ADR-005 記載のとおり 1 件あたり数分）が無駄に重複する。

現状は単一ユーザー MVP（`infra/deploy.sh` が全ジョブに `USER_ID=default` を焼き込み）であり、クロスユーザー衝突は実際にはまだ発生しない。本 ADR は Phase 2 の多ユーザー化に向けた設計判断を先に固定するものである。なお `DIFFICULTY` もジョブ env で全ユーザー共通のため、多ユーザー化時は同一記事をスターした 2 人の `cache_key` が **必ず衝突** する点に注意。

本 ADR では 2 つの設計判断を扱う：
1. キャッシュ層のアーキテクチャ
2. 同一 `cache_key` の同時生成が起きたときの **待機者調整**（後発ジョブの per-user Podcast をどう作るか）

## 検討した選択肢

### 判断 1: キャッシュ層アーキテクチャ

| 選択肢 | メリット | デメリット |
|---|---|---|
| **共有キャッシュ層 `podcastCache/{cache_key}` + per-user `Podcast` 参照** | 配信経路（`get_podcasts_for_user` は `user_id` 絞り込み、`get_podcast` は所有権検査）と iOS クライアント契約を変えずに済む。重い処理（Gemini/TTS/音声保存）だけ共有化し、安価な per-user メタ doc は残す | per-user doc とキャッシュ doc の二層になる |
| 単一の共有 Podcast を全ユーザーで 1 ドキュメント共有 | 完全な単一実体 | 配信・所有権・一覧順序のロジックを全面改修、iOS 契約にも波及。現段階で過剰 |

### 判断 2: 待機者調整（`processing` を踏んだ後発ジョブの扱い）

| 選択肢 | メリット | デメリット |
|---|---|---|
| **B: 今回スキップ + 次回トリガーで補完** | 実装が `continue` のみで極小。失敗時は次回実行が `failed` を見て再生成する自己修復。waiters 引き継ぎロジック不要 | 後発ユーザーへの提供が次回トリガーまで遅延（鮮度を犠牲。コスト削減目的には影響しない） |
| A: waiters 方式（生成側が代理作成） | 1 回の生成で全レーサーが即時に成果物を受け取れる（UX 最良） | doc 直列化トランザクション + `processing` リース（生成側クラッシュ時の固着対策）+ 失敗時引き継ぎが必須でコード/テストが増える |

## 決定

### 判断 1
**共有キャッシュ層 + per-user `Podcast` 参照** を採用する。

- 新コレクション `podcastCache/{cache_key}`（`cache_key = f"{article_id}__{difficulty}__{language}"`、doc-id 直引きで O(1)）。
  - フィールド: `article_id` / `difficulty` / `language` / `audio_url`(共有 blob) / `japanese_intro_text` / `duration_seconds` / `status`(`processing`|`completed`|`failed`) / `created_at`。
- 共有音声は決定論的パス `podcasts/cache/{cache_key}.mp3` に保存する。
- per-user `Podcast` は従来どおり作成するが `audio_url` は共有 blob を指し、`japanese_intro_text`/`duration_seconds` はキャッシュから複写（denormalize）する。配信・署名付き URL 生成・所有権チェックは無改修で動く。
- `language` は言語切替が未実装のためキー次元として用意するが当面は定数（例 `"ja-en"` = 日本語イントロ + 英語本文）。

### 判断 2
**MVP は B（今回スキップ + 次回補完）で開始し、必要になったら A（waiters）へ拡張する。**

ジョブはスター記事ごとに次を行う：
1. 既存の per-user 重複チェックでユーザーが当該 Podcast を持てばスキップ（冪等性維持）。
2. `podcastCache[cache_key]` を見る。
   - `completed` → Gemini/TTS を呼ばず per-user `Podcast` を作成（キャッシュヒット）。
   - 無し / `failed` → トランザクションで `processing` を原子的に確保（`try_acquire_job_lock` と同じ read→条件付き write）。成功したら生成 → `completed` 書き込み → per-user `Podcast` 作成。
   - `processing`（他ジョブが生成中）→ **その記事を今回はスキップ**。何も作らない。

スキップされた記事は `starred_article_ids` に残るため、**B の次回 `podcast-generator` 実行**（B 自身の次のスター/dismiss 操作による起動、または定時実行）でキャッシュヒットとして補完される。

## 理由

- 配信経路と iOS 契約を変えずに、目的（重複する Gemini/TTS の排除）を確実に達成できるため、共有キャッシュ + per-user 参照を採る。
- 現在の規模（単一ユーザー MVP、Phase 2 で多ユーザー）では即時性より単純さ・堅牢さが勝る。B は実装が最小で、生成側クラッシュ時も次回実行が `failed` を見て再生成する自己修復性を持つ。B の弱点は「後発ユーザーへの提供遅延」だが、これは鮮度の問題でありコスト削減という主目的を損なわない。
- A は per-user 難易度（Phase 2）が入り、かつ後発ユーザーへの即時提供を要件化した段階で、`processing` リース込みで設計するのが妥当。MVP で先取りするとトランザクション設計・失敗引き継ぎのコストに見合わない。

## 結果（影響）

- 補完の主経路は **B 自身の次回 `podcast-generator` 起動**。スキップ記事は `starred_article_ids` に残り、その後の任意のジョブ実行が全 starred を再走査して拾うため、補完は「B が将来何らかの操作をする」ことに自然に依存する。
- 定時 `podcast-generator` 実行（`infra/deploy.sh`、毎朝 07:00）はバックストップとなる。ただし現状の Scheduler は単一の焼き込み `USER_ID` で `:run` するため、**多ユーザー化時は per-user fan-out（ユーザー列挙して各 USER_ID で起動）が別途必要**になる。これは本 ADR の範囲外の Phase 2 インフラ課題として残す。
- 後発ユーザーが `processing` を踏んだ場合、その記事の Podcast 提供は次回トリガーまで遅延しうる（許容する）。
- 生成失敗は `status=failed` で残し、次回実行が再確保・再生成する。`completed` キャッシュは TTL で勝手に失効させない（参照され続ける限り保持する共有成果物）。
- 記事本文が `content_fetched_at` で再取得されても、MVP では「取得後は不変」とみなしキャッシュを陳腐化対象としない。将来は content hash をキーへ加える案。
