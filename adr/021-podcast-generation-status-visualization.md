# ADR-021: Podcast 生成状態の可視化（processing 行 + status API + ポーリング）

**ステータス:** 採用済み  
**決定日:** 2026-06-25  
**対象読者:** バックエンド / Web / iOS 開発者  
**関連:** [ADR-011](011-podcast-generation-status.md)（本 ADR が更新・実現）・[ADR-005](005-job-trigger-on-action.md)・[ADR-006](006-cross-user-podcast-cache.md)・[ADR-023](023-tts-partial-failure-skip-join.md)（partial_failed の generator 側実装）・`backend/api/routers/articles.py`・`backend/jobs/podcast_generator/main.py`・backend PR #24/#25（issue #38）

> **補足（2026-06-25・issue #41）:** 本 ADR は status 値 `partial_failed` を定義したが、当時 generator は `completed`/`failed` のみ書いていた。TTS 一部失敗時にスキップ結合して `partial_failed` を実際に記録する generator 側の実装は [ADR-023](023-tts-partial-failure-skip-join.md) で採用済み。

## 背景（ADR-011 の Phase 2 実現）

ADR-011 では「生成中（processing）エピソードの可視化」を Phase 2 課題として保留した。理由は、当時の API レスポンスに `status` フィールドが含まれておらず、実現にはバックエンド変更が必須だったためである。

issue #38 では、実装コストと可用性のバランスを取り直し、**Star 操作時に即座に processing 行を Firestore に保存し、API レスポンスに status を公開し、Web/iOS クライアントがポーリングで生成状態を表示する**実装を採用した。

## 決定

### A. Star エンドポイント（/articles/:id/star）の改定

**契約変更:**
- **旧:** `POST /articles/:id/star` → 200 OK、記事を star_articles に追加
- **新:** `POST /articles/:id/star` → **202 Accepted**、per-user Podcast 行を `status="processing"` で原子的に保存

**実装:**
- `api/routers/articles.py` の `try_acquire_user_podcast` トランザクションで、Firestore に `Podcast` ドキュメントを作成（`status="processing", article_id, user_id, difficulty, type, created_at`）
- `ActionResponse(status="processing", podcast_id)` で 202 を返す
- 非幸せパス（ユーザーが既にキャッシュを所有など）は 200 OK で即座に完了を返す

### B. PodcastResponse スキーマの拡張

`backend/api/schemas.py` に status・error_message を追加：

```python
class PodcastResponse(BaseModel):
    id: str
    article_id: str
    user_id: str
    difficulty: str
    type: str
    intro: str
    intro_jp: str
    # 新規フィールド:
    status: str  # "processing" | "completed" | "failed" | "partial_failed"
    error_message: str | None = None  # TTS 失敗時のメッセージ
    # 既存フィールド:
    audio_url: str | None = None  # processing 中は None
    duration: int | None = None
    created_at: str  # ISO 8601
    updated_at: str
```

**影響:**
- GET /podcasts・GET /podcasts/:id が status を返すため、Web/iOS で生成中をポーリング表示できる
- 既存クライアント（status を無視）も互換性がある（新フィールド追加のみ）

### C. Firestore スキーマの status 複合インデックス

生成中の Podcast を dedup するため、`podcasts` コレクションに status を含めた複合インデックスを追加：

```
podcasts (user_id, difficulty, type, status, article_ids)
```

- **purpose:** `podcast_exists_for_article(user_id, article_id)` で既存キャッシュを検査する際、`status="processing"` は別扱い（重複を許容、並列生成を防ぐ）
- **deployment:** `backend/scripts/deploy_firestore_indexes.sh`（冪等スクリプト）で適用。本番環境への先行デプロイが必要（API デプロイ前）

### D. 生成状態遷移フロー

```
Podcast.status の遷移:
  processing
    ├→ (生成成功) completed ← audio_url・duration セット、error_message=null
    ├→ (全セリフ失敗) failed ← error_message セット
    └→ (一部失敗) partial_failed ← audio_url セット、error_message セット

  - job/podcast_generator/main.py の promote_user_podcast(status, error_message=...)で遷移
  - 遷移は update_at を更新して一度限り
```

### E. Web クライアント（StatusBadge + ポーリング）

**新規コンポーネント:**
- `web/components/ui/StatusBadge.tsx`：status に応じた UI（生成中は spinner、完了は✓、失敗は ⚠）
- `web/hooks/usePodcastListPolling.ts`：5 秒間隔ポーリング（最大 120 秒）で processing → completed/failed を検知

**配置:**
- `/podcast` 一覧ページで各カード上に StatusBadge を表示
- Star 直後から processing に遷移する
- ユーザーが `/feed` に戻ってポーリング継続（UI 不完全だが仕様）

### F. iOS クライアント（status API 消費準備）

API は status を提供済みだが、iOS 側の生成中ポーリング表示は **ios#15** として今後対応（ADR-011 の phase 2）。

- `Podcast` モデルは `status` フィールドを持つ
- 現段階では status を表示せず、pull-to-refresh で完成エピソードを反映する挙動を継続

## 理由

1. **即時フィードバック**: `status="processing"` を即座に保存・返却することで、ユーザーが Star 後すぐに「生成中」を認識できる
2. **重複生成の排除**: 複合インデックスで `status=processing` を含めた dedup により、並列生成リスクを低減
3. **拡張性**: Web Push（ADR-020）と組み合わせて「生成完了通知」を提供可能
4. **バックエンド変更最小**: API 契約（202 + status フィールド）とインデックスの追加のみで実現

## 結果（影響）

### API 契約の変更

| エンドポイント | 変更 |
|---|---|
| `POST /articles/:id/star` | 200 → **202 Accepted**（契約変更）。レスポンス本体で `status="processing"` を返す |
| `GET /podcasts` | 既存。レスポンスに `status` フィールド追加（新フィールド、既存値との互換） |
| `GET /podcasts/:id` | 既存。`status` フィールド追加 |

### Firestore スキーマ

- 新規複合インデックス: `podcasts(user_id, difficulty, type, status, article_ids)` 
- **デプロイ前提**: Cloud Firestore へのインデックス適用が必要（`deploy_firestore_indexes.sh` で自動化）

### Web 実装

- 新規: `components/ui/StatusBadge.tsx`・`hooks/usePodcastListPolling.ts`
- 修正: `/podcast` ページで StatusBadge + ポーリング統合

### テスト・検証

**受け入れ条件（issue #38 実装完了時点）:**
1. Star 直後に `GET /podcasts` で status=processing が見える
2. ジョブ完了後に status=completed へ遷移
3. Web ポーリング UI が processing → completed を反映
4. インデックス適用後、重複生成テストが緑

### iOS 側の未対応

- API は `status` を提供済み（本 ADR）
- iOS アプリの「生成中ポーリング UI」は iOS#15 として別途スコープ（pull-to-refresh で完成を表示する現仕様を継続）

## トレードオフと留保事項

### Web ポーリングの限界

- 5 秒間隔のポーリングは、ユーザーが常に Podcast タブを見ている必要がない
- `/feed` に遷移後、後ほど `/podcast` に戻るとポーリングが再開される（完成をキャッチ）
- 完全にリアルタイムな通知は Web Push（ADR-020）の併用が理想

### iOS ポーリング未実装

- 本 ADR では API のみ提供。iOS クライアント側のポーリング表示は ios#15 で対応
- 当面 iOS は pull-to-refresh で完成を確認する（Web パリティの範囲）

### 202 Accepted による API 契約変更

- 既存クライアント（status 未対応）でも動作するが、状態表示が失われる（明示的な対応が必要）
- backend PR #24 で全テストが緑になるまでカバレッジを検証

</content>
</invoke>