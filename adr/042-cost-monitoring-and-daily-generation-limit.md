# ADR-042: コスト監視（予算アラート）とユーザー別日次生成上限

**ステータス:** 採用済み
**決定日:** 2026-06-30
**対象読者:** バックエンド / インフラ / Web / iOS 開発者
**関連:** [ADR-005](005-job-trigger-on-action.md)（Star→生成トリガ）・[ADR-006](006-cross-user-podcast-cache.md)（クロスユーザーキャッシュ）・[ADR-015](015-audit-logging.md)（監査ログ）・[ADR-018](018-generic-api-rate-limiting.md)（汎用レート制限）・[PRD §6/§10](../prd/2026-05-31-news-listen.md)・GitHub issue #82

## 背景

PRD の非機能要件はコスト「月次 5USD 以下」（PRD §6/§10）。汎用レート制限（ADR-018）で Star の
TTS 濫用は一定抑止しているが、**予算超過の検知（Billing アラート）とユーザー別の生成上限が無い**ため、
コスト SLO を運用上担保できていない。設定ミスや想定外利用で TTS / Gemini コストが膨らむリスクを塞ぐ。

決定論点は (A) ハードなコスト上限の担保手段、(B) ユーザー別生成上限の計数位置とリセット方式。

## 検討した選択肢

### (A) コスト上限の担保

| 選択肢 | メリット | デメリット |
|---|---|---|
| アプリ側の生成上限のみ | 追加インフラ不要 | 集計漏れ・抜け穴があると青天井。真の請求額を見ていない |
| **Cloud Billing Budget + 閾値アラート**（採用） | GCP 請求額そのものを監視し、閾値（50/90/100%）超過で通知。アプリ実装と独立した最終防衛線 | 課金アカウント ID が必要・通知到達はメール/Pub/Sub 設定依存 |

### (B) ユーザー別生成上限の計数

| 選択肢 | メリット | デメリット |
|---|---|---|
| ロール窓（直近 24h） | 連続性 | 「1 日」境界が曖昧。doc が単一で長期保持 |
| **日次バケット（UTC 日付を doc-id に含める）**（採用） | 暗黙リセット（新しい日＝新 doc）。recommendations と同方式 | 過去日 doc が蓄積（極小・TTL は将来検討） |
| 計数位置: ジョブ側（実生成直前） | コスト厳密 | 同期的な明示拒否（429・理由）を返せない（AC 違反） |
| **計数位置: star エンドポイント（新規 per-user 予約時）**（採用） | 同期的に 429＋理由を返せる（AC 充足） | star 時点判定のため厳密なコスト一致ではない（best-effort） |

## 決定

- **(A) Cloud Billing Budget を採用**。`infra/setup.sh` に予算＋閾値アラートの作成を追加（`billingbudgets.googleapis.com`、
  `GCP_BILLING_ACCOUNT_ID` 変数で opt-in、既定予算 5USD/月）。これを**コスト上限の最終防衛線（ハード）**とする。
- **(B) ユーザー別日次生成上限を star エンドポイントで判定**。`FirestoreClient.consume_daily_generation(user_id, max_per_day)`
  がコレクション `dailyGenerationCounts`、doc-id `f"{user_id}_{day}"`（UTC 日付）でトランザクション計数する。
  `try_acquire_user_podcast` が新規予約（非 None）を返したときのみ消費し、超過時は予約をリリースして
  **429 + Retry-After（翌 UTC 0 時まで）+ 理由**を同期返却し、当該 star を記録しない。
- 上限は環境変数 `PODCAST_DAILY_LIMIT_PER_USER`（既定 20・0 で無効）で設定可能（`.env.example` と一致）。
- 上限到達は監査ログ `generation_limit_reached`（ADR-015）に記録する。

## 理由

- 真のコスト保証は GCP 請求額の監視（Billing Budget）でしか担保できない。アプリ側上限は**補助的な best-effort soft cap**と
  位置づけ、二層で守る。
- AC「超過時は明示的に拒否し理由を返す」を満たすには同期経路（star エンドポイント）での 429 が必要。ジョブ側の厳密計数は
  この要件と両立しないため採らない。
- 日次バケット doc-id はリセットを TTL/cron 無しで実現でき、既存の recommendations と一貫する。

## 結果（影響・既知の限界）

- **backend:** `consume_daily_generation` 追加、`star_article` に上限ゲート、`AuditAction` に `generation_limit_reached`、
  `.env.example` に `PODCAST_DAILY_LIMIT_PER_USER`。境界値・初回・新日付リセット・retry_after の自動テスト。
- **infra:** `infra/setup.sh` に「Cloud Billing Budget + 予算アラート」セクション、`infra/deploy.sh` の env-vars に
  `PODCAST_DAILY_LIMIT_PER_USER` を追加。
- **クライアント（web/ios）:** 429 受領時に「本日の上限に達しました（次回可能時刻）」を表示（フォローアップ）。
- **既知の限界（best-effort soft cap ゆえの非厳密性。ハード上限は Billing Budget で担保）:**
  - cross-user キャッシュヒット（ADR-006）は実コスト 0 でもクォータを消費する（保守的）。
  - 既に `failed` 行が残る記事の再 star は計数されない（稀な抜け穴）。
  - 同一記事の同時 star が上限境界で競合すると、最大 1 回分だけ上限を超過しうる（極小・スケール上許容）。
  - `dailyGenerationCounts` の過去日 doc は自然消滅しない（蓄積は極小。Firestore TTL は将来検討）。
