# ADR-018: 汎用 API レート制限

**ステータス:** 採用済み
**決定日:** 2026-06-24
**対象読者:** バックエンド / Web / iOS 開発者
**関連:** [ADR-014](014-login-rate-limiting.md)（ログイン試行レート制限）・[ADR-016](016-cors-and-security-headers.md)（CORS・セキュリティヘッダ／純関数ポリシー+薄い適用シェルの先例）・`backend/api/ratelimit.py`・`backend/shared/firestore_client.py::consume_rate_limit`

## 背景

ADR-014 でログイン試行のレート制限（`loginAttempts` コレクション + トランザクション）が確立された。これを一般化して、**API 全体（`/auth`、`/feed`、`/articles`、`/podcasts`、`/settings`、`/admin`）と特定エンドポイント（`POST /articles/{id}/star` など）に対して統一的な固定ウィンドウレート制限を適用する必要がある。

攻撃者が大量のリクエストを送信して DB 負荷を高めたり、リソース計算コスト（TTS による Podcast 生成など）を消費させる DoS 攻撃を軽減するため、以下を目標とする：

- **2 層制御**: user 単位（セッション由来）+ IP 単位（ハッシュ化）で、多角的な制限を実施
- **既定値テーブル**: バケット（`api`、`star` 等）ごとに既定上限を持つ
- **環境変数パラメータ化**: 本番・ステージング・ローカル開発で柔軟に調整可能
- **max=0 での無効化**: テスト・開発環境で制限を外せる
- **効率的な計数**: Firestore トランザクション + 固定ウィンドウ

## 検討した選択肢

| 選択肢 | メリット | デメリット |
|---|---|---|
| **Firestore 汎用カウンタ + 2層Depends**（採用） | 既存スタック。複数インスタンス耐性。ADR-014と同型 | リクエスト毎に read/write（高頻度API） |
| API ゲートウェイレート制限（Cloud Armor） | インスタンス前処理。Firestore 不要 | 追加インフラ・GCP 依存・設定複雑度 |
| middleware + slowapi | FastAPI 標準パターン | メモリ内 → インスタンス毎・再デプロイでリセット（ADR-014の教訓） |

## 決定

### A. Firestore スキーマ：汎用 `rateLimits` コレクション

```
rateLimits/{doc_id}:
{
  "key": "api:ip:{hash}",       // or "api:user:{user_id}" など
  "count": int,                  // ウィンドウ内カウント
  "window_start": timestamp      // ウィンドウ開始時刻（失効判定用）
}
```

- `loginAttempts` とは独立した新規コレクション（目的別分離）
- `key` フィールドはレート制限キー（バケット + 軸情報）
- `locked_until` フィールドは不要（汎用 API では一時ロック不要、429 でいい）

### B. データベースメソッド：`consume_rate_limit`

`shared/firestore_client.py` に追加：

```python
def consume_rate_limit(
    self,
    key: str,
    now: datetime | None = None,
    max_requests: int = 0,
    window_seconds: int = 60,
) -> tuple[bool, int]:
    """汎用レート制限。固定ウィンドウ内のカウンタを原子的に更新する。
    
    返り値: (allowed: bool, retry_after: int)
      - allowed=True: リクエスト許可。retry_after=0。
      - allowed=False: 超過。retry_after=窓終了までの秒数（最小1）。
    """
```

**仕様**：
- `max_requests <= 0` → 無効化。即座に (True, 0) を返す（DB アクセス無し）
- 初回（doc 未存在）→ count=1, window_start=now でリセット、allowed=True
- ウィンドウ内で count < max_requests → count +1 して保存、allowed=True
- ウィンドウ失効（経過秒数 >= window_seconds） → count=1, window_start=now でリセット、allowed=True
- count >= max_requests → 超過試行。**doc 更新せず（非カウント）**、allowed=False、retry_after 計算
- **境界条件**: 経過秒数 == window_seconds のとき失効扱い（>= 判定）
- **トランザクション**: ADR-014 に倣い `@firestore.transactional` で read→write を原子化

### C. ポリシー層：`evaluate_rate_limit`

`api/ratelimit.py` に純関数として定義：

```python
def evaluate_rate_limit(
    allowed_user: bool,
    retry_user: int,
    allowed_ip: bool,
    retry_ip: int,
) -> tuple[bool, int]:
    """両軸の判定を統合する（DB 非依存）。
    
    いずれかが NotAllowed → 全体 NotAllowed。
    retry_after = max(各軸の値)。
    """
```

**利点**：
- ロジックを Firestore から分離（テストが簡単）
- user/ip の組み合わせを柔軟に制御可能

### D. 依存関数：`rate_limit(bucket: str)`

`api/ratelimit.py` にファクトリとして実装：

```python
def rate_limit(bucket: str):
    """バケット名（"api", "star" 等）ごとのレート制限依存関数を生成。"""
    async def _rate_limit_dep(request: Request, db: Depends(...)) -> None:
        # 環境変数を毎回読み込む（importlib.reload 不要）
        max_requests = _env_int(f"{bucket.upper()}_RATELIMIT_MAX_REQUESTS", ...)
        window_seconds = _env_int(f"{bucket.upper()}_RATELIMIT_WINDOW_SECONDS", ...)
        
        # max_requests <= 0 で no-op
        # 有効時: IP + user の両軸で計数
        # evaluate_rate_limit で統合
        # NotAllowed なら HTTPException(429) + Retry-After
    return _rate_limit_dep
```

**特性**：
- リクエスト毎に env を読む → 再起動なしに設定変更可能
- user_id 解決失敗（未認証等）も安全に処理 → IP 軸のみに fallback
- トークン詳細をログ・エラー本文に含めない

### E. 既定値テーブル

```python
_DEFAULTS = {
    "api": (120, 60),      # API グローバル: 120 req/60sec = 2 req/sec
    "star": (10, 3600),    # Star エンドポイント: 10 req/1hour（計算負荷高い）
}
```

環境変数で上書き可能：
- `API_RATELIMIT_MAX_REQUESTS`・`API_RATELIMIT_WINDOW_SECONDS`
- `STAR_RATELIMIT_MAX_REQUESTS`・`STAR_RATELIMIT_WINDOW_SECONDS`

未知バケットは既定 (0, 0) + warning ログ。

### F. 配線

**`api/main.py`**：
```python
app.include_router(
    feed.router,
    dependencies=[Security(verify_api_key), Depends(rate_limit("api"))]
)
# 同様に articles, podcasts, settings, admin
```

**`api/routers/articles.py`**：
```python
@router.post("/articles/{article_id}/star", ...)
def star_article(
    ...,
    _rl: None = Depends(rate_limit("star")),  # star 専用制限
):
    pass  # dismiss は API 全体制限のみ（専用制限なし）
```

## 理由

- **Firestore トランザクション**: 複数インスタンス下でも整合性保証。ADR-014 の実績を活用。
- **2層制御**: user 軸でユーザー単位の濫用を検出。IP 軸で同一ネットワークからの低速 DoS を検出。両立で耐性向上。
- **環境変数パラメータ化**: 本番（120/60）・ステージング（更に厳格）・開発（0で無効）を切り替え容易。
- **max=0 無効化**: テスト・開発で制限無しで動作確認可能（既存テスト互換性保証）。
- **超過試行の非カウント**: カウント値の一貫性を保つ。制限中のクライアントの再試行が他のユーザーを巻き込まない。
- **Retry-After ヘッダ**: HTTP 仕様準拠。クライアント側で待機時間を認識して自動 backoff 可能。

## 結果（影響）

### スキーマ・データベース
- Firestore 新規コレクション `rateLimits/{doc_id}`。TTL ポリシーは将来設定可（古いレコード自動削除）。

### API
- **全エンドポイント** (`/auth`, `/feed`, `/articles`, `/podcasts`, `/settings`, `/admin`): `Depends(rate_limit("api"))` で保護（issue #37「全API共通」要件。`/auth/login` は ADR-014 専用ロックも併走するが api 上限はそれより緩いため先に専用ロックが効く）
- **`POST /articles/{id}/star`**: 追加で `Depends(rate_limit("star"))` 適用（計算負荷が高いため専用制限）
- **`POST /articles/{id}/dismiss`**: グローバル API 制限のみ（計算軽量）

### 環境変数
- `.env.example` に新規追加：
  ```
  API_RATELIMIT_MAX_REQUESTS=120
  API_RATELIMIT_WINDOW_SECONDS=60
  STAR_RATELIMIT_MAX_REQUESTS=10
  STAR_RATELIMIT_WINDOW_SECONDS=3600
  ```

### エラーレスポンス
- 429 Too Many Requests
- ボディ: `{ "detail": "Too many requests. Please try again later." }`
- ヘッダ: `Retry-After: {秒数}`
- 内部詳細（トークン・IP・user_id）は含めない

### テスト
- `conftest.py` の `patch.dict` に `API_RATELIMIT_MAX_REQUESTS=0`, `STAR_RATELIMIT_MAX_REQUESTS=0` を追加
  → 既存全テストが制限無しで実行（互換性維持）
- 新規テストモジュール：
  - `tests/test_firestore_rate_limit.py`: consume_rate_limit 単体
  - `tests/test_ratelimit_policy.py`: evaluate_rate_limit 単体
  - `tests/test_ratelimit_dependency.py`: 依存関数の env・例外処理
  - `tests/test_api_articles.py` 追記: star の 429・ジョブ非起動検証

## トレードオフと留保事項

### Firestore リード・ライトコスト
- リクエスト毎に consume_rate_limit で read/write 1 回（user 軸は条件付き）
- 見積: 1日 10K API リクエスト（複数ユーザー） → 月 300K read/write。Firestore 無料枠（50K/日）を超過
- **対応**: 本番運用で超過した場合、Redis 導入を新 ADR で上書き可能。現在は Firestore スタックで統一

### XFF 信頼（ADR-014 継承）
- `X-Forwarded-For` クライアント由来で詐称可能
- Cloud Run を通す場合、信頼できるプロキシが右側値を付与。本実装は最左 1 個（直接接続元）採用
- 詐称の影響を user 軸カウントで緩和

### 同一 IP 複数ユーザー（ADR-014 継承）
- 家族 WiFi 環境では 1 人の失敗が他者をカウント消費
- 既定値 120/60 なら影響軽微（同一 IP 5 人の場合でも 1 人/秒以上のリクエストペース）

### 固定ウィンドウの簡潔性
- スライディングウィンドウ（直近 N 秒のイベント数を常時計数）ではなく、簡潔な固定ウィンドウ実装
- 実務的には十分（ADR-014 実績）

### 2 軸計数の非原子性（ADR-014 継承）
- 1 リクエストは IP 軸 → user 軸の順に `consume_rate_limit` を**個別トランザクション**で 2 回呼ぶ。キー内の read→write は原子的だが、**2 キー間は非原子**（ADR-014 の `register_failed_login` を 2 キーに呼ぶのと同方針）。
- 結果、IP 軸は許可だが user 軸で 429 になるケースでは、IP 軸カウンタは既に 1 消費済み（ロールバックしない）。すなわち拒否されたリクエストでも IP 軸が前進しうる。
- **影響は fail-safe**: 過大計上のみ（実際よりやや厳しめ＝コスト保護として安全側）で、過小計上（すり抜け）は起きない。補償減算の複雑化を避けるため、この割り切りを許容する。

### 不変条件
- **api 上限は login ロックアウト (ADR-014) より緩く設定**: ログインが先に引っかかり、API 制限は後発。ADR-014 の `LOGIN_RATELIMIT_MAX_ATTEMPTS=5, WINDOW=900` なら、10 回/1時間程度の login 失敗で 15 分ロック。その間、API リクエストは送信不可なので、API 上限（120/60）の検証順序に依存しない。

## 関連する今後の課題

- **Redis 導入**: Firestore コスト超過時は Redis キャッシュを前置き（新 ADR）
- **バースト許容**: Token Bucket 方式の検討（現在は固定ウィンドウ）
- **キーロテーション**: IP ハッシュ用 salt の定期更新（セキュリティ強化）
