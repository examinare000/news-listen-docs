# ADR-020: 生成完了通知の送信基盤と Web Push（VAPID）

**ステータス:** 採用済み
**決定日:** 2026-06-25
**対象読者:** バックエンド / Web 開発者
**関連:** [ADR-011](011-podcast-generation-status.md)（生成ステータス管理）・`backend/shared/notifier.py`・`backend/api/routers/notifications.py`・`backend/jobs/podcast_generator/main.py`・`web/public/sw.js`・`web/hooks/useWebPushSubscription.ts`・backend PR #27 / web PR #10（issue #39）

## 背景

podcast 生成完了（[ADR-011](011-podcast-generation-status.md)）の際、ユーザーに即座に通知したい。当初フェーズ2想定だった「生成完了通知」を、**Web Push（VAPID）を用いて前倒し実装**し、将来 iOS APNs（ios#15）を並列実装する拡張性を確保する。

### 要件

1. **非同期性**: ジョブ完了直後、ブラウザに対して Web Push で通知（httpOnly Cookie では JavaScript が受信できないため、別チャネルとして実装）
2. **購読管理**: ユーザーが Web Push 受信を ON/OFF 切り替え可能
3. **失効対応**: 購読期限切れ（410 Gone など）を送信時に自動掃除
4. **拡張性**: `Notifier` Protocol で `WebPushNotifier` と将来 `ApnsNotifier` を実装可能

## 検討した選択肢

| 選択肢 | メリット | デメリット |
|---|---|---|
| **Web Push（VAPID）（採用）** | 標準 W3C 仕様。サーバー側で鍵管理。モダンブラウザ対応 | 購読 API（PushManager）は HTTPS + Secure Context 必須。旧ブラウザ非対応 |
| Email 通知 | 標準的。広く対応 | 迅速性なし。低頻度向け |
| WebSocket（Server-Sent Event） | リアルタイム。双方向 | 接続維持のコスト。モバイルブラウザで電力消費増 |
| iOS APNs | 即座性。ネイティブ統合 | iOS のみ。別鍵管理・証明書更新の運用 |

## 決定

### A. 送信基盤設計（`shared/notifier.py`）

**Protocol ベースの抽象**で、実装を差し替え可能にする：

```python
class Notifier(Protocol):
    def notify_completion(
        self,
        user_id: str,
        *,
        title: str,
        body: str,
        data: dict[str, Any] | None = None,
    ) -> None: ...
```

- **`WebPushNotifier`**: pywebpush で Web Push 送信
- **`NoOpNotifier`**: no-op（VAPID 未設定時、ローカル・テスト安全）
- **将来 `ApnsNotifier`**: iOS APNs 向け実装（ios#15）

**構築**（`build_notifier` ファクトリ）:
- VAPID 公開鍵・秘密鍵・メールが全て環境変数に設定されていれば → `WebPushNotifier`
- 1つでも未設定なら → `NoOpNotifier`（no-op モード。ログに鍵値は出さない）

### B. Firestore スキーマ：`pushSubscriptions`

```
pushSubscriptions/{doc_id}:
{
  "user_id": "user123",
  "endpoint": "https://fcm.googleapis.com/...",
  "p256dh": "...",      // W3C PushSubscription.keys.p256dh
  "auth": "...",        // W3C PushSubscription.keys.auth
  "created_at": timestamp
}
```

- **doc_id**: `sha256(endpoint)` で一意に。エンドポイント URL は長いため hash 化
- **冪等性**: endpoint が同じなら upsert（同一デバイスから複数回登録しても上書き）
- **失効掃除**: 送信時に 404/410 エラーなら自動削除（`delete_push_subscription`）

### C. API エンドポイント（`api/routers/notifications.py`）

#### `GET /notifications/vapid-public-key`
- **概要**: VAPID 公開鍵を返す
- **リクエスト**: パラメータなし
- **レスポンス**: `{ "public_key": "..." }`
- **未設定時**: 404 Not Found（クライアントが機能検出）

#### `POST /notifications/subscriptions`
- **概要**: Web Push 購読を登録（冪等）
- **認証**: セッション必須（`get_user_id` 依存）
- **リクエストボディ**: W3C 標準形式
  ```json
  {
    "endpoint": "https://fcm.googleapis.com/...",
    "keys": {
      "p256dh": "...",
      "auth": "..."
    },
    "expirationTime": null  // 受け取るが無視（スケーラビリティ）
  }
  ```
- **レスポンス**: 201 Created、`{ "status": "subscribed" }`
- **冪等性**: endpoint が既に存在すれば上書き（upsert）

#### `DELETE /notifications/subscriptions?endpoint=...`
- **概要**: Web Push 購読を解除（冪等）
- **認証**: セッション必須
- **クエリパラメータ**: `endpoint` の URL
- **レスポンス**: 200 OK、`{ "status": "unsubscribed" }`
- **冪等性**: endpoint が不在でも 200

### D. 送信トリガー（バックエンド）

**ジョブ完了時** (`backend/jobs/podcast_generator/main.py`)

```python
# 1. Podcast 生成完了
# 2. Firestore に status=completed を保存
# 3. インライン送信（非非同期）
notifier.notify_completion(
    user_id=user_id,
    title="Podcast 完了",
    body="記事「...」の生成が完了しました",
    data={
        "url": f"https://example.com/podcasts/{podcast_id}",
        "podcast_id": podcast_id,
        "article_id": article_id,
    }
)
# 4. 送信失敗しても warning のみ（非致命）。ジョブ status=completed で終了
```

**WHY インライン送信**:
- ジョブ内で直接送信（キューに入れない）
- 失敗は warning のみで非致命（ジョブは completed で成功）
- 購読がなければ no-op（コスト 0）
- VAPID 未設定のローカル・テストでも no-op で安全

### E. Web 受信側（`web/`）

#### Service Worker（`public/sw.js`）
```javascript
self.addEventListener('push', (event) => {
  const data = event.data.json();
  self.registration.showNotification(data.title, {
    body: data.body,
    icon: 'icon-192.png',
    badge: 'badge-96.png',
    data: data,  // url, podcast_id など
  });
});

self.addEventListener('notificationclick', (event) => {
  event.notification.close();
  event.waitUntil(
    clients.matchAll().then((clientList) => {
      // 該当 URL へ遷移
      const url = event.notification.data.url || '/';
      return clients.openWindow(url);
    })
  );
});
```

#### PushManager Hook（`hooks/useWebPushSubscription.ts`）
- 状態機械: Idle → Requesting → Granted/Denied/Unavailable
- 機能検出: `Notification` / `ServiceWorkerContainer` / `PushManager` の順
- 権限拒否時: グレースフルに機能を無効化（UI に警告 UI）
- 購読トグル: ON/OFF で `/notifications/subscriptions` の POST/DELETE

#### 設定画面UI
- Push 通知 ON/OFF トグル
- 有効/無効状態の表示
- 権限拒否時は「ブラウザ設定で有効化してください」メッセージ

#### 関連ファイル
- `lib/pushBrowserPort.ts`: PushManager 抽象化
- `lib/webpush.ts`: 購読 API（POST/DELETE）の fetch wrapper

### F. 環境変数（`.env.example` に追記）

```
# Web Push (VAPID)
VAPID_PUBLIC_KEY=
VAPID_PRIVATE_KEY=
VAPID_CLAIMS_EMAIL=example@example.com
```

- **VAPID_PUBLIC_KEY / VAPID_PRIVATE_KEY**: `web-push generate-vapid-keys` で生成
- **VAPID_CLAIMS_EMAIL**: VAPID ペイロード内の `sub` フィールド（マニュアル）
- **鍵の実値**: `.env.example` には書かない（実値は本番環境のみ Secret Manager へ）

### G. Firestore セキュリティルール

```
match /pushSubscriptions/{doc_id} {
  allow create, update, delete: if request.auth.uid != null && request.auth.uid == resource.data.user_id;
  allow read: if request.auth.uid != null && request.auth.uid == resource.data.user_id;
}
```

- セッション認証が必須（`request.auth.uid`）
- 自分自身の購読のみ操作可能

## 理由

- **Web Push（VAPID）**: W3C 標準。サーバー側鍵管理で、クライアント毎の証明書更新が不要。複数デバイス対応で、iOS・Web を別々に管理可能（拡張性）。
- **Protocol 抽象**: 将来 APNs を追加する際、`Notifier` インターフェースを実装するだけで実現可能。既存ジョブコードは変更不要。
- **インライン送信**: ジョブキューに入れるより簡潔で、失敗が非致命のため。購読なければ呼び出しコストも 0。
- **購読冪等性**: デバイス登録をリトライしても一度限りの副作用（upsert で安全）。
- **失効掃除**: 410/404 で購読を自動削除。次回以降の送信コストを削減（ゴミ購読を溜めない）。

## 結果（影響）

### スキーマ

- **新規 Firestore コレクション**: `pushSubscriptions/{doc_id}`（user_id + endpoint の関連）

### API

- **新規エンドポイント**:
  - `GET /notifications/vapid-public-key` → public_key 返却 / 未設定 404
  - `POST /notifications/subscriptions` → 購読登録、201 Created
  - `DELETE /notifications/subscriptions?endpoint=...` → 購読削除、200 OK

- **既存エンドポイント**: 変更なし

### 環境変数（backend）

```
VAPID_PUBLIC_KEY=
VAPID_PRIVATE_KEY=
VAPID_CLAIMS_EMAIL=contact@example.com
```

### Web

- **新規**: `hooks/useWebPushSubscription.ts`（Hook）、`lib/webpush.ts`（fetch API）、`lib/pushBrowserPort.ts`（PushManager 抽象）
- **修正**: `public/sw.js`（push イベント受信）、`public/manifest.json`（Service Worker 登録）、設定画面に購読トグル追加
- **design asset**: `web/public/icon-192.png`（通知アイコン） — 現在未配置（follow-up タスク）

### テスト

- `backend/tests/test_notifier.py`: Protocol / WebPushNotifier / NoOpNotifier 単体
- `backend/tests/test_api_notifications.py`: エンドポイント（VAPID キー取得・購読 POST・DELETE）、冪等性、失効掃除
- `web/src/hooks/__tests__/useWebPushSubscription.test.ts`: Hook 状態機械・権限処理・UI 連動

### セキュリティ面での改善

- **認証**: 全エンドポイントでセッション必須（`get_user_id`）
- **冪等性**: endpoint hash による重複排除で、リトライ安全
- **失効掃除**: 410/404 で購読削除。ゴミレコード による Firestore 汚染防止

### 既存との互換性

- **VAPID 未設定**: NoOpNotifier で no-op。既存テスト・ローカル開発は無変更で動作
- **Web**: Service Worker 登録時に `GET /vapid-public-key` で機能検出。404 なら購読 UI を無効化（グレースフル）

### 残課題・トレードオフ

- **通知アイコン**: `web/public/icon-192.png` は デザインアセット提供待ち（本 ADR は API・購読機構のみ）
- **iOS APNs**: 別個の実装（ios#15）。ただし Protocol 抽象で統合可能
- **購読の永続期間**: Firestore TTL ポリシーは将来設定可（自動クリーンアップ）

## トレードオフと留保事項

### Web Push の限界

- **HTTPS + Secure Context 必須**: ローカル開発（http://localhost）では動作するが、本番環境では HTTPS が必須。
- **購読エンドポイント長寿命**: 一度購読登録されたエンドポイントは、クライアントが破棄するまで有効。デバイス破棄時はサーバー側で自動掃除されない（失効検出時に削除）。

### VAPID 鍵運用

- 秘密鍵は Secret Manager で管理。環境変数経由でコンテナへ注入。
- 鍵ローテーション時は、古い購読への送信が失敗（410）になり、自動削除される。

### 購読リスト容量

- 1 ユーザーが複数デバイス・ブラウザを持つ場合、endpoint 数が増える。
- 見積: 1 ユーザー 10 デバイス程度を想定（家族利用）。Firestore コスト許容範囲内。

### iOS との連携

- 本 ADR は **Web Push のみ**。iOS は APNs（ios#15）で別実装。
- Web と iOS で通知が重複する可能性。クライアント側で「既に通知受け取った」を検知する場合は、プッシュペイロードに `id` を含める。

