# ADR-046: クライアントエラー/クラッシュの集約（受信口 `/client-errors` への一元化）

**ステータス:** 採用済み
**決定日:** 2026-06-30
**対象読者:** バックエンド / Web / iOS 開発者
**関連:** [ADR-043](043-error-observability.md)（エラー可観測性・構造化ログ/スクラブの土台）・[ADR-031](031-web-e2e-and-error-boundary.md)（web エラーバウンダリ）・[ADR-037](037-gateway-api-key-client-distribution.md)（ゲートウェイ X-API-Key）・GitHub issue #83

## 背景

ADR-043 で backend／infra のエラー可観測性（構造化ログ・スクラブ・SLO メトリクス・アラート）は整備したが、
**クライアント（web/iOS）側で起きたエラー・クラッシュ**は収集経路が無く、PRD の「アプリクラッシュ率 1% 以下」
（PRD §2）を測定できない。ADR-043 の「結果」でも web の `error.tsx` フック・iOS の MetricKit を follow-up として
予告していた。これを実装するにあたり、(A) 収集の集約先、(B) web の捕捉経路、(C) iOS の捕捉経路を決める。

方針の制約: 1〜5 人規模ゆえ外部ベンダー依存・課金は避け、ADR-043 の既存資産（scrub・構造化ログ・Cloud Logging）を再利用する。

## 検討した選択肢

### (A) 収集の集約先

| 選択肢 | メリット | デメリット |
|---|---|---|
| Sentry / Crashlytics 等の外部 SaaS | 高機能・自動グルーピング | 依存・課金・PII 管理が増。SDK 同梱。1〜5 人に過剰 |
| **backend 受信口 `POST /client-errors` → 既存の構造化ログ→Cloud Logging**（採用） | 依存ゼロ。scrub/severity/アラートを ADR-043 と共用。web/iOS を一元化 | 自前の受信口・グルーピングは log ベース |
| クライアントから直接 Cloud Logging へ | 経由が減る | クライアントに GCP 資格情報が要る（重大なリスク）。不可 |

### (B) web の捕捉経路

| 選択肢 | メリット | デメリット |
|---|---|---|
| **React エラーバウンダリ（`error.tsx`/`global-error.tsx`）＋ `window` の `error`/`unhandledrejection`**（採用） | レンダリング例外と非同期/イベント例外の両方を捕捉 | 2 経路の薄い実装が要る |
| エラーバウンダリのみ | 単純 | ハンドラ外の async・イベント由来を取りこぼす |

### (C) iOS の捕捉経路

| 選択肢 | メリット | デメリット |
|---|---|---|
| **MetricKit（`MXMetricManagerSubscriber`）のクラッシュ診断**（採用） | OS 標準・追加 SDK 不要・次回起動時に確実配信 | リアルタイムでない（次回起動時）。診断は端末側で生成 |
| サードパーティ Crash SDK | リアルタイム | 依存・課金・dSYM 管理 |

## 決定

- **(A)** backend に `POST /client-errors` を追加（`api/routers/client_errors.py`）。`ClientErrorReport{source, kind, message?, context?}` を
  受け、`logger.error("client_error", extra=...)` で記録（ADR-043 の ScrubbingFormatter が送出時に PII/秘密を伏せる）。
  **認証セッション不要**（エラーは未ログイン時にも起き得る）。保護は **X-API-Key（ゲートウェイ・ADR-037）＋ レート制限**。
  未ログインでも通すため **CSRF 免除**（`CSRF_EXEMPT_PATHS` 既定に `/client-errors` を追加）。受理応答は 202。
- **(B web)** `lib/reportClientError.ts` が BFF 経由で最小ペイロードを POST（送信失敗は握りつぶし＝報告で本処理を壊さない・`keepalive`）。
  `error.tsx`/`global-error.tsx` は `useEffect` で render エラーを報告。**非漏洩契約は維持**（message/stack/digest を UI/console に出さず、
  送信先は監視基盤のみ。backend が scrub）。`components/ClientErrorReporter.tsx` を layout にマウントし `window` の
  `error`/`unhandledrejection` を購読。
- **(C iOS)** `Observability/CrashReporter.swift`（`MXMetricManagerSubscriber`）を `NewsListenAppApp.init()` で購読登録。
  受領したクラッシュ診断は**純粋関数 `CrashReportFormatter`** で原始値（exception type/code・signal・terminationReason 切詰・build version）
  のみに整形して `/client-errors` へ送る。自由記述（`virtualMemoryRegionInfo` 等）は PII 懸念のため送らない。送信用 APIClient は
  セッショントークン無し（X-API-Key のみ）。

## 理由

- 既存の scrub・構造化ログ・Cloud Logging・アラート（ADR-043）に相乗りでき、web/iOS のエラーを 1 経路に集約できる。外部 SaaS の依存・課金・PII 管理を避けられる。
- web は 2 経路（バウンダリ＋window）でレンダリング/非同期の双方を捕捉。iOS は OS 標準の MetricKit で追加 SDK・dSYM 管理を増やさない。
- `MXCrashDiagnostic` はテスト生成不可のため、スクラブ/整形を純粋関数に分離して単体テスト可能にした（MetricKit 配信経路はディバイス依存で薄く保つ）。
- `/client-errors` を未認証・CSRF 免除にする代わり、X-API-Key とレート制限・最小スキーマ・上限長で濫用を抑える。

## 結果（影響）

- **backend:** `api/routers/client_errors.py`（202・構造化ログ）、`api/main.py` 登録、`api/middleware/csrf.py` と `.env.example` の
  CSRF 既定免除に `/client-errors` 追加。受信/スクラブ依存/無認証/バリデーション/CSRF 免除のテスト。
- **web:** `lib/reportClientError.ts`・`components/ClientErrorReporter.tsx`・`app/error.tsx`/`global-error.tsx` のフック実装（非漏洩維持）。送信・購読・非漏洩のテスト。
- **iOS:** `Observability/CrashReporter.swift`・`ClientErrorPayload.swift`、`Networking` の `.clientErrors`＋`reportClientError`、`NewsListenAppApp.init()` 登録。純粋整形関数と送信のテスト。
- **運用:** クライアント由来エラーは `event=client_error` で Cloud Logging に出る。Error Reporting 連携はログ形状から自動取り込み可能（infra 任意）。
