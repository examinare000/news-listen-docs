# ADR-043: エラー可観測性（構造化ログ・スクラブ・生成所要時間メトリクス・アラート）

**ステータス:** 採用済み
**決定日:** 2026-06-30
**対象読者:** バックエンド / インフラ / Web / iOS 開発者
**関連:** [ADR-015](015-audit-logging.md)（監査ログ）・[ADR-031](031-web-e2e-and-error-boundary.md)（web エラーバウンダリ）・[ADR-042](042-cost-monitoring-and-daily-generation-limit.md)（Billing アラートの先例）・[PRD §2/§6](../prd/2026-05-31-news-listen.md)・GitHub issue #83

## 背景

PRD の成功指標に「アプリクラッシュ率 1% 以下」「Star → 生成完了 2 分以内」（PRD §2/§6）があるが、
これらを測定・検知する仕組みが無い。エラーや SLO 逸脱に気づく手段が無く、品質目標が運用上未担保。
可観測性の土台を入れて障害検知と SLO 監視を可能にする。一方、ログ/エラーに資格情報・PII を
出さない方針（`agent-rules/12`）は継続する。

決定論点は (A) ログの構造化とスクラブの方式、(B) 生成所要時間 SLO の信号源、(C) アラートの実現手段。

## 検討した選択肢

### (A) ログ構造化・機密スクラブ

| 選択肢 | メリット | デメリット |
|---|---|---|
| 外部 APM（Datadog 等）導入 | 高機能 | 依存・課金増。1〜5 人規模に過剰 |
| **stdlib logging + JSON フォーマッタ + フォーマッタ段階スクラブ**（採用） | 依存ゼロ。Cloud Run の stdout→Cloud Logging に乗る。severity 解釈可 | 自前のスクラブ正規表現の保守 |
| Filter でレコードを書き換えてスクラブ | 実装が単純 | 共有 LogRecord を破壊し caplog 等を汚染。例外トレースを取りこぼす |

### (B) 生成所要時間の信号源

| 選択肢 | メリット | デメリット |
|---|---|---|
| Cloud Monitoring カスタムメトリクス API | 厳密 | 書き込み API 呼び出しコスト・複雑 |
| **構造化ログ行（emit_metric）+ log-based metric**（採用） | 追加 API 不要。既存ログ経路に乗る | 集計は log-based metric 経由（粒度は分） |

### (C) アラート

| 選択肢 | メリット | デメリット |
|---|---|---|
| **log-based metric + Cloud Monitoring アラートポリシー + メール通知チャネル**（採用・ADR-042 と同パターン） | GCP ネイティブ・IaC（setup.sh）で冪等作成 | gcloud alpha 依存・課金アカウント/通知先設定が要 |

## 決定

- **(A)** `shared/logging_config.py` を追加。`configure_logging()` が stdout ハンドラに
  **スクラブするフォーマッタ**を設定（`LOG_FORMAT=json` で Cloud Logging 互換 JsonFormatter、
  既定はテキストの ScrubbingFormatter）。スクラブは**フォーマッタ段階**で行い LogRecord を
  書き換えない（メッセージ本文だけでなく**例外トレースも**スクラブ、caplog 非汚染）。
  対象: `key=value`（refresh_token 等の複合キー含む）・ヘッダ/クッキーの `Key: value`・
  Bearer トークン・メールアドレス。一般語の自然文（記事タイトル等）は壊さない。
- グローバル例外ハンドラ（`api/main.py` の `Exception` ハンドラ）が未捕捉 500 を構造化ログに記録し、
  本文には内部情報を漏らさない定型メッセージのみ返す。
- **(B)** `emit_metric()` で `podcast_generation_duration`（status・duration_ms・article_id）を
  生成成功/失敗時に emit。`emit_metric` はテレメトリ失敗で本処理を壊さない（例外安全・予約属性退避）。
- **(C)** `infra/setup.sh` に「Cloud Monitoring アラート」セクションを追加（`GCP_ALERT_EMAIL` で opt-in）。
  生成失敗の log-based metric・メール通知チャネル・アラートポリシーを冪等に作成。

## 理由

- 1〜5 人規模では stdlib + Cloud Logging で十分。外部 APM の依存・課金は便益に見合わない。
- スクラブをフォーマッタ段階に寄せることで「出力されるものだけ」を確実に伏せ、レコード破壊による
  テスト汚染・例外トレース取りこぼしを避ける（実装・レビューで顕在化した落とし穴を回避）。
- 生成所要時間は構造化ログ→log-based metric が最小コスト。2 分 SLO（PRD §2/§6）の逸脱検知に使える。

## 結果（影響）

- **backend:** `shared/logging_config.py`（scrub / ScrubbingFormatter / JsonFormatter /
  configure_logging / emit_metric）、`api/main.py` のグローバル例外ハンドラ、
  `jobs/podcast_generator/main.py` の所要時間メトリクス。スクラブ（複合キー・散文非破壊・
  例外トレース）・例外ハンドラ（ASGI 結合）・メトリクス emit の自動テスト。
- **infra:** `infra/setup.sh` に監視セクション（log-based metric + 通知チャネル + アラートポリシー、
  `monitoring.googleapis.com` 有効化）。
- **環境変数:** `LOG_FORMAT`（json で構造化）・`LOG_LEVEL`（不正値は INFO）・`GCP_ALERT_EMAIL`（通知先）。
- **web / iOS（フォローアップ）:** web は `app/error.tsx` / `global-error.tsx` の「監視基盤へ送る」
  フックを実装（非漏洩契約維持）、iOS は `NewsListenAppApp.init()` で MetricKit によるクラッシュ収集を初期化。
