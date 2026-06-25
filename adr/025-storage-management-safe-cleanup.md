# ADR-025: ストレージ使用量レポートと安全な一括削除（共有キャッシュ blob 非削除）

**ステータス:** 採用済み
**決定日:** 2026-06-25
**対象読者:** バックエンド開発者 / 運用担当者
**関連:** [ADR-006](006-cross-user-podcast-cache.md)（クロスユーザーキャッシュ）・[ADR-024](024-daily-digest-generation.md)（digest は per-user blob）・[ADR-015](015-audit-logging.md)（監査）・`backend/api/routers/podcasts.py`・`backend/api/storage_cleanup.py`・issue #47（PRD §5.3 P1）

## 背景

PRD §5.3 P1 は「キャッシュ済み音声のサイズ確認・一括削除」を求める。backend には使用量レポートも削除エンドポイントも無かった。生成音声は GCS Lifecycle で 30 日後に自動削除される（PRD §6）が、ユーザーが**自分のライブラリの使用量を確認し能動的に整理する**手段が無い。

削除には固有の危険がある。single Podcast の音声は**クロスユーザー共有キャッシュ blob**（`podcasts/cache/{cache_key}.mp3`・[ADR-006](006-cross-user-podcast-cache.md)）を指し、**あるユーザーがこれを削除すると同じ記事×難易度を持つ他ユーザーが再生不能**になる。一方 digest（[ADR-024](024-daily-digest-generation.md)）は per-user blob で共有されない。

## 検討した選択肢

| 選択肢 | メリット | デメリット |
|---|---|---|
| **doc は全削除・blob は digest(per-user)のみ削除・共有 blob は Lifecycle 任せ**（採用） | 共有キャッシュを壊さない。実装が単純で安全不変条件が1関数に集約 | 一括削除しても共有 blob の物理バイトは即時解放されない（30日 GC 待ち） |
| 共有 blob も参照カウントを見て削除 | 物理解放が即時 | 参照カウント管理が複雑・競合・誤削除リスク。YAGNI |
| blob を一切削除しない（doc のみ） | 最も安全 | digest の per-user blob が孤児として 30 日残る（小さな無駄） |

## 決定

### 使用量レポート `GET /podcasts/storage/usage`（read-only・監査なし）
- ユーザーが**所有する Podcast** を取得し、各 `audio_url` の GCS `blob.size` 実測を合計して返す（`total_bytes`・`podcast_count`・per-item）。
- `StorageClient.get_blob_size(blob_name)` は blob 不在・空名・取得失敗時に **0**（read 安全）。失敗は warning ログで可観測化。

### 一括削除 `POST /podcasts/storage/cleanup`（要認証・監査あり）
- リクエスト `older_than_days: int | None`（None=全所有 Podcast、整数 N=`created_at < now - N日`）。
- **安全不変条件（最重要）**: 削除可否は純粋関数 `is_blob_deletable(podcast)` に集約する。
  - `type != "digest"` → blob 削除しない（single/レガシは共有または共有相当）。
  - `audio_url` が共有キャッシュ prefix `podcasts/cache/` で始まる → 削除しない（二重防御）。
  - digest の per-user blob のみ削除可。
- **Firestore の per-user Podcast doc は type を問わず削除**する（doc が残り blob だけ消えた「壊れ Podcast」を避ける）。共有 blob は触れず 30 日 Lifecycle GC に委ねる。
- **集計の正直さ**: `delete_blob` は成否を bool で返し、`deleted_blob_count`・`freed_bytes` は**実際に削除成功した blob のみ**計上する。blob 削除は best-effort（失敗してもループ継続・doc 削除は確実化）。
- 監査（[ADR-015](015-audit-logging.md)）: 新 `AuditAction = "storage_cleanup"` を成功後に記録。`details` は件数・サイズ・`older_than_days` のみで、**Podcast id・blob パス・article_id は含めない**。

## 理由

- 共有 blob を絶対に削除しないことで、一括削除によるクロスユーザーの再生不能事故を**構造的に**防ぐ。安全判定を I/O から切り離した純粋関数に集約し、`type=single` の blob 削除呼び出しが存在しないことを単体テスト＋API テストの二重で固定する。
- 参照カウント管理は複雑さに見合わない（YAGNI）。共有 blob は Lifecycle の 30 日 GC が回収するため、即時物理解放を諦めても運用上の損失は小さい。
- 集計を実削除成功分に限定し、削除失敗を「解放済み」と誤報告しない。

## 結果（影響）

- 新エンドポイント 2 本（`api/routers/podcasts.py`）。純粋関数 `api/storage_cleanup.py`（`is_blob_deletable` / `select_podcasts_to_delete`）。`StorageClient` に `get_blob_size` / `delete_blob`（bool）、`FirestoreClient` に `delete_podcast`（冪等）。`AuditAction` に `storage_cleanup`。
- クライアントは使用量表示・一括削除 UI を本 API で実装できる（UI は web/ios の別 issue）。
- 受け入れ条件: 使用量合計・blob 不在 0・cleanup の doc 削除・**共有 blob 非削除**・digest blob 削除・older_than フィルタ・監査・冪等をテストで検証（`tests/test_storage_cleanup.py`・`tests/test_api_podcasts.py`）。

## トレードオフと留保事項

- **物理解放の遅延**: 一括削除しても共有 blob の GCS バイトは即時解放されない（30 日 Lifecycle 任せ）。使用量レポートは「所有 Podcast のサイズ」を測るため、doc 削除でレポート値は下がる。
- **N+1**: 使用量は所有 Podcast 件数分 `blob.size` を呼ぶ（read-only・低頻度のため許容。将来コスト化すれば `Podcast.size_bytes` の denormalize を検討）。
- **個別削除は非提供**: 本 issue は「一括削除」。単体 Podcast 削除 API は将来課題。
