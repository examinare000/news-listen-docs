# ADR-023: TTS 一部失敗のスキップ結合とリトライ（partial_failed の生成側実装）

**ステータス:** 採用済み
**決定日:** 2026-06-25
**対象読者:** バックエンド開発者
**関連:** [ADR-021](021-podcast-generation-status-visualization.md)（status 可視化／本 ADR が generator 側を実現）・[ADR-006](006-cross-user-podcast-cache.md)（クロスユーザーキャッシュ）・`backend/jobs/podcast_generator/tts_generator.py`・`backend/jobs/podcast_generator/main.py`・issue #41（PRD §6 P0・§12）

## 背景

[ADR-021](021-podcast-generation-status-visualization.md) は `PodcastStatus` に `partial_failed` を定義し status 遷移図に「一部失敗 → partial_failed」を記したが、**generator 側は未実装**で `completed` / `failed` の二値しか書いていなかった。Podcast は 2 セグメント（日本語イントロ＝JP voice / 英語本編＝EN voice）を個別に TTS し PCM を連結するが、いずれか 1 回の TTS 失敗で**全体が failed** になり、せっかく成功したセグメントも破棄されていた。PRD §6 P0・§12 は「TTS 並列で一部セリフ失敗→リトライ後もスキップ結合し partial_failed 記録」を求める。

## 検討した選択肢

| 選択肢 | メリット | デメリット |
|---|---|---|
| 現状維持（一部失敗＝全体 failed） | 単純 | 成功セグメントを捨て、ユーザーに何も届かない。受け入れ条件を満たせない |
| **セグメント単位リトライ＋失敗スキップ結合＋partial_failed 記録** | 成功分の音声が完成し配信できる。瞬間的障害をリトライで救済 | 結果型・分岐の追加。partial 時のキャッシュ／ステータス整合の設計が要る |
| 英文を文単位に細分化して TTS | 失敗粒度が細かい | 音質・プロソディ・コスト影響大。受け入れ条件に不要（別 issue） |

## 決定

セグメント粒度（JP イントロ / EN 本編の 2 つ）で**リトライ付きスキップ結合**を実装する。

- **リトライ**: 各セグメントを `max_attempts`（既定 2）まで再試行する。リトライ対象は TTS 応答構造不正を表す `ValueError` のみ（それ以外の想定外例外は即送出して上位の失敗処理に委ねる）。プロセス内バックオフ（スリープ）は持たない（上位に「次回トリガーで再生成」する自己修復層があるため）。
- **スキップ結合**: 成功セグメントのみ順序維持で連結する。`TtsGenerator.generate_audio` は結果値オブジェクト `TtsResult(audio, failed_segments, error_message)` を返す。全成功なら `failed_segments=[]`、一部失敗（成功≥1）なら失敗セグメント名を保持、**全失敗は例外** `TtsGenerationError` を送出（上位の既存 failed 経路が処理）。
- **per-user Podcast**: 一部失敗時は成功分の音声 blob を参照する `partial_failed` 行を保存し、`error_message` に失敗セグメント名のみを記録する（形式 `"TTS failed for segments: english_body"`。資格情報・スタックトレース・外部 API レスポンスは含めない）。
- **キャッシュ（ADR-006）**: 一部失敗時の共有キャッシュは `failed` に留める。`CacheStatus` は 3 値（`processing`/`completed`/`failed`）のままで `partial_failed` を持ち込まない。
- **通知**: 部分失敗でも音声は完成しているため、完了通知を送る（文面で「一部失敗」を明示）。

## 理由

- **成功分を捨てない**: 1 セグメント失敗で全体を失わせず、ユーザーに視聴可能な音声を届ける（P0 体験要件）。
- **瞬間的障害の救済**: TTS の応答揺らぎ（safety filter 等）に対し同一プロセス内リトライが有効。バックオフは上位の再生成層があるため不要。
- **キャッシュ非汚染と自己修復**: 部分音声を `completed` で共有キャッシュに載せると後続ユーザーが欠落音声を cache hit で受け取る。`failed` に留めれば次回トリガーで再確保・フル生成が走り、完全成功すれば `completed` へ自己修復する。これは ADR-006 のキャッシュ意味論（completed のみ共有配布）と既存の failed 自己修復方針に整合する。
- **status 値の非対称は意図的**: キャッシュは「共有成果物として不完全＝failed」、per-user Podcast は「このユーザーには部分提供＝partial_failed」を表す。両者の関心が異なるため別値が正しい。

## 結果（影響）

- `PodcastStatus` の `partial_failed` が generator から実際に書かれるようになった（定義のみ→実装）。ADR-021 の status 遷移図が generator 側でも成立する。
- 受け入れ条件: 一部 TTS 失敗を注入したテストで「成功分の音声が完成」かつ「per-user `status=="partial_failed"` と `error_message` 記録」を検証（`tests/test_tts_generator.py`・`tests/test_podcast_generator_main.py`）。
- 既知の留保: `partial_failed` 行は `promote_user_podcast` が processing からのみ遷移する仕様上、再生成でフル成功しても per-user の status は `partial_failed` のまま据え置かれる（同一キャッシュ blob が上書きされ音声自体は完全化しうる）。`failed` 行と同じ既存の挙動であり、再昇格は別途検討。
- スコープ外: TTS の**並列化**（PRD §12「並列」）と英文の文単位細分化は本 ADR に含めない（順次のまま・別 issue）。
