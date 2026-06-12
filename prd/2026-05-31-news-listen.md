# PRD: news-listen — 海外テックニュース自動収集・Podcast化アプリ

**バージョン:** 1.4  
**作成日:** 2026-05-31  
**最終更新:** 2026-06-10  
**ステータス:** 実装中（バックエンドコード実装完了・テスト 90件パス / GCP セットアップ完了 / Cloud Run デプロイ完了 / iOS アプリ未着手）  
**想定ユーザー数:** 1〜5人（個人・ファミリー利用）

---

## 目次

1. [背景・課題](#1-背景課題)
2. [目的と成功指標](#2-目的と成功指標)
3. [ターゲットユーザー](#3-ターゲットユーザー)
4. [プラットフォーム](#4-プラットフォーム)
5. [機能要件](#5-機能要件)
6. [非機能要件](#6-非機能要件)
7. [システムアーキテクチャ](#7-システムアーキテクチャ)
8. [データモデル](#8-データモデル)
9. [Podcast スクリプト生成仕様](#9-podcast-スクリプト生成仕様)
10. [コスト見積もり](#10-コスト見積もり)
11. [MVP スコープ](#11-mvp-スコープ)
12. [設計決定事項](#12-設計決定事項)
13. [未解決事項](#13-未解決事項)

---

## 1. 背景・課題

エンジニアにとって HackerNews・Zenn.dev 等の海外テックニュースを毎日読み続けることは難しい。課題は2つある。

1. **情報量の多さ**: 毎日大量に流れるニュースの中から自分に関係する記事を見つけるのに時間がかかる。
2. **言語コスト**: 英語記事の精読には時間とエネルギーを要し、継続が難しい。

本アプリは「通勤・家事中に耳でキャッチアップ」というユースケースに特化し、AI によるパーソナライズと音声化で両課題を同時に解決する。

---

## 2. 目的と成功指標

### 目的

| # | 目的 | 詳細 |
|---|------|------|
| 1 | 英語リスニング練習 | 記事の日本語概要を先に聴いてから英語本編を聴くことで、内容を理解した状態でリスニング練習できる |
| 2 | テックニュースキャッチアップ | 海外・国内テックニュースを毎日自動収集し、パーソナライズされた順序で効率的に確認できる |
| 3 | 文脈把握 | Star した記事だけでなく関連ニュースも含めて生成することで、背景情報ごと把握できる |

### 成功指標（MVP）

| 指標 | 目標値 |
|------|--------|
| Podcast 生成時間（Star 操作から完了まで） | 2分以内 |
| 日次バッチ完了時刻 | 06:30 JST まで（06:00 開始から30分以内） |
| 月次ランニングコスト | 5USD 以下（1ユーザー・1日5エピソード相当） |
| アプリクラッシュ率 | 1% 以下 |

---

## 3. ターゲットユーザー

- **プライマリ**: 英語学習中のエンジニア（TOEIC 600〜900 相当）、毎日のニュースキャッチアップを効率化したい
- **セカンダリ**: 同一家族・友人グループでの共有利用（最大4名まで同一バックエンドを共有）

---

## 4. プラットフォーム

| レイヤー | 技術選定 |
|----------|---------|
| iOS アプリ | SwiftUI ネイティブ、iOS 17以上 |
| バックエンド | Google Cloud（Cloud Run + Cloud Scheduler + Cloud Tasks + Firestore + Cloud Storage） |
| AI / TTS | Gemini 2.5 Flash（スクリプト生成）+ OpenAI TTS / Google Cloud TTS（音声合成） |

---

## 5. 機能要件

優先度凡例: **P0** = MVP必須 / **P1** = MVP後フェーズ2 / **P2** = 将来検討

### 5.1 Feed タブ（ニュース一覧）

| 機能 | 詳細 | 優先度 |
|------|------|--------|
| RSS 購読 | 初期ソース：HackerNews・Zenn.dev。アプリ内から任意 RSS URL を追加・削除可能 | P0 |
| 記事情報表示 | タイトル・ソース名・公開日時・推定スコアを表示 | P0 |
| スワイプ操作 | 右スワイプ → Star（Podcast 生成キューに即時追加）、左スワイプ → Dismiss（非表示） | P0 |
| 既読管理 | Dismiss 済み記事は翌日以降も表示しない | P0 |
| パーソナライズレコメンド | Star/Dismiss 履歴を Gemini 2.5 Flash（Context Caching）で分析し、関心スコア順に記事を並べて毎日表示 | P0 |
| 検索 | キーワードで取得済み記事を全文検索 | P2 |

### 5.2 Podcast タブ（音声再生）

| 機能 | 詳細 | 優先度 |
|------|------|--------|
| エピソード一覧 | タイトル・日付・再生時間・難易度・ステータス（生成中/完了/失敗）を表示 | P0 |
| 単体記事エピソード | Star した記事1件につき1エピソードをオンデマンド生成（Star後1〜2分で完了） | P0 |
| 日本語イントロ | 各エピソード冒頭に日本語で日付・記事概要（1〜5センテンス）を読み上げ | P0 |
| 英語本編 | 記事本文＋関連ニュースを英語で掛け合い（男性話者A・女性話者B）形式で読み上げ | P0 |
| 難易度選択 | 6段階（下記 [9節](#9-podcast-スクリプト生成仕様) 参照） | P0 |
| 再生速度調整 | x0.5〜x2.5 の8段階（AVPlayer の rate 機能で実装） | P0 |
| 生成進捗表示 | 生成中のエピソードはプログレスインジケーターを表示し、完了時にプッシュ通知 | P0 |
| 再生位置保存 | アプリを閉じても秒単位で再生位置を記憶（Firestore + ローカルキャッシュ） | P0 |
| 日次ダイジェスト | その日のおすすめ記事（Star + 高スコア記事）をまとめた1本を毎朝自動生成 | P1 |
| オフライン再生 | 生成済み音声ファイルをデバイスにキャッシュし、オフラインでも再生可能 | P1 |

#### ⚡ Podcast 生成フロー（詳細）

```
Star 操作
  │
  ▼
Cloud Run Service (REST API)
  │  受付後即座に202 Acceptedを返す
  ▼
Cloud Tasks キューへタスク投入
  │
  ▼
[バックグラウンド処理]
  ├─ 1. 記事本文フェッチ（newspaper3k / RSS全文）
  ├─ 2. 関連ニュース検索（コーパス内タイトル類似度）
  ├─ 3. Gemini 2.5 Flash でスクリプト生成（構造化JSONで出力）
  ├─ 4. セリフ単位でOpenAI TTS / Google TTSへ並列リクエスト
  │      話者A → 男性ボイス
  │      話者B → 女性ボイス
  ├─ 5. FFmpeg で音声結合 → MP3化
  └─ 6. Cloud Storage に保存 → Firestore ステータス更新 → Push通知

所要時間目安: 1〜2分（5分番組相当）
```

### 5.3 Settings タブ（設定）

| 機能 | 詳細 | 優先度 |
|------|------|--------|
| RSS ソース管理 | 購読中のソース一覧表示・新規 URL 追加・削除 | P0 |
| デフォルト難易度 | Podcast 生成時のデフォルト難易度を設定 | P0 |
| デフォルト再生速度 | 再生速度のデフォルト値を設定 | P0 |
| 日次ダイジェスト設定 | ON/OFF、対象記事数（3〜10件） | P1 |
| ストレージ管理 | キャッシュ済み音声のサイズ確認・一括削除 | P1 |

---

## 6. 非機能要件

| 項目 | 要件 |
|------|------|
| **可用性** | バッチ処理は毎日 06:00 JST に自動実行。06:30 JST までに完了すること |
| **レスポンス** | Star 操作後 2分以内に Podcast 生成完了（Cloud Tasks 非同期処理） |
| **コスト** | 月次 5USD 以下（1ユーザー・1日5エピソード相当）。詳細は [10節](#10-コスト見積もり) 参照 |
| **セキュリティ** | iOS アプリ → バックエンド通信は固定 API キー（GCP Secret Manager 管理）で認証。iOS アプリに API キーをハードコードしない |
| **データ保持** | 生成音声ファイルは 30日後に Cloud Storage Lifecycle ルールで自動削除 |
| **エラーハンドリング** | TTS 並列処理で一部セリフが失敗した場合、リトライ後も失敗したセリフはスキップして結合。Firestore のステータスを `partial_failed` で記録 |
| **スケーラビリティ** | 個人利用（〜5人）前提。Cloud Tasks のレートリミット設定で TTS API のレート超過を防ぐ |

---

## 7. システムアーキテクチャ

### 7.1 処理フロー全体

```
[定時バッチ] 毎日 06:00 JST (Cloud Scheduler)
  │
  ├─ rss-fetcher-job (Cloud Run Jobs)
  │    RSS取得 → 本文フェッチ → Firestore に保存
  │
  └─ recommendation-job (Cloud Run Jobs)
       Gemini (Context Caching) で履歴分析 → 関心スコア再計算 → Firestore に保存

[オンデマンド] iOS App → Cloud Run Service (REST API)
  │
  ├─ POST /articles/:id/star
  │    → Cloud Tasks にPodcast生成タスクを投入 → 202 Accepted 返却
  │
  └─ Cloud Tasks Worker (Cloud Run Service)
       スクリプト生成 → 並列TTS → 音声結合 → Storage保存 → Push通知
```

### 7.2 使用サービス一覧

| サービス | 用途 | 選定理由 |
|:---------|:-----|:---------|
| Cloud Run Jobs | RSS フェッチ・レコメンド計算バッチ | マネージド・スケールゼロ。バッチ向けコンテナ実行 |
| Cloud Run Service | REST API サーバ・TTS ワーカー | FFmpeg 等の任意バイナリをコンテナ化して実行可能 |
| Cloud Scheduler | 定時バッチ起動 | Cloud Run Jobs との統合が簡単 |
| Cloud Tasks | TTS 並列処理制御 | レートリミット・リトライをマネージドで管理 |
| Gemini 2.5 Flash | スクリプト生成・レコメンド | 高速・低コスト。Context Caching で日次履歴分析コストを削減 |
| OpenAI TTS | 音声合成（メイン） | $15/1M文字。複数話者の自然な演じ分けが可能 |
| Google Cloud TTS | 音声合成（フォールバック） | OpenAI TTS 障害時の代替 |
| Firestore | データストア | サーバーレス。個人利用規模は無料枠内に収まる |
| Cloud Storage | 音声ファイル保存 | Lifecycle ルールで30日後自動削除 |
| Secret Manager | API キー管理 | バックエンドの環境変数に安全に注入 |

### 7.3 API エンドポイント（主要）

| メソッド | パス | 説明 |
|----------|------|------|
| GET | `/feed` | レコメンド済み記事一覧取得 |
| POST | `/articles/:id/star` | 記事を Star → Podcast 生成キュー投入 |
| POST | `/articles/:id/dismiss` | 記事を Dismiss |
| GET | `/podcasts` | エピソード一覧取得 |
| GET | `/podcasts/:id` | エピソード詳細・音声URL取得 |
| PATCH | `/podcasts/:id/position` | 再生位置更新 |
| GET | `/settings` | ユーザー設定取得 |
| PUT | `/settings` | ユーザー設定更新 |

---

## 8. データモデル

### articles

```json
{
  "id": "string",
  "title": "string",
  "url": "string",
  "source": "string",              // "hackernews" | "zenn" | カスタムURL
  "content": "string",             // フェッチ済み本文
  "publishedAt": "timestamp",
  "fetchedAt": "timestamp",
  "contentFetchedAt": "timestamp | null"
}
```

### userPrefs（ユーザーごと）

```json
{
  "userId": "string",
  "starredArticleIds": "string[]",
  "dismissedArticleIds": "string[]",
  "rssSources": [
    { "name": "string", "url": "string", "enabled": true }
  ],
  "defaultDifficulty": "string",   // "toeic_600" | "toeic_730" | "ielts_5" | "ielts_7" | "eiken_2" | "eiken_pre1"
  "defaultPlaybackSpeed": 1.0,
  "digestEnabled": true,
  "digestArticleCount": 5          // 3〜10
}
```

### recommendations（日次・ユーザーごと）

```json
{
  "userId": "string",
  "date": "string",                // "2026-05-31"
  "rankedArticles": [
    { "articleId": "string", "score": 0.95 }
  ],
  "generatedAt": "timestamp"
}
```

### podcasts

```json
{
  "id": "string",
  "type": "single" | "digest",
  "articleIds": "string[]",
  "difficulty": "string",
  "audioUrl": "string | null",     // Cloud Storage URL（完了後に設定）
  "japaneseIntroText": "string",
  "durationSeconds": 300,
  "status": "processing" | "completed" | "failed" | "partial_failed",
  "errorMessage": "string | null",
  "playbackPositionSeconds": 0,    // 再生位置（秒）
  "createdAt": "timestamp",
  "completedAt": "timestamp | null",
  "userId": "string"
}
```

---

## 9. Podcast スクリプト生成仕様

### 構成

| パート | 長さ | 内容 |
|--------|------|------|
| 日本語イントロ | 30〜60秒 | 日付読み上げ → 記事タイトルと概要（1〜5センテンス）→ 「英語でお聴きください」等の導入フレーズ |
| 英語本編 | 3〜10分 | 記事本文の要約・解説 + 関連ニュースの補足情報。話者A（男性）・話者B（女性）の掛け合い形式 |

### 難易度ガイドライン

| レベル | 語彙 | 文構造 | 話速 |
|--------|------|--------|------|
| TOEIC 600以下 | 中学〜高校基本語彙 | 短文・単純構造 | ゆっくり明瞭 |
| TOEIC 730〜900 | ビジネス基本語彙 | 複合文あり | 標準 |
| IELTS 5.5〜6.5 | アカデミック語彙（入門） | 接続詞・節あり | 標準〜やや速め |
| IELTS 7.0以上 | 高度な学術・専門語彙 | 複雑な構造 | ネイティブ速度 |
| 英検2級 | 日常〜社会的話題語彙 | 標準的な英文 | 標準 |
| 英検準1級以上 | 時事・専門語彙 | 論説文レベル | やや速め |

### スクリプト出力フォーマット（Gemini への指示）

Gemini にはセリフ単位の構造化 JSON を出力させ、そのまま TTS API へ投入する。

```json
{
  "intro": {
    "language": "ja",
    "text": "..."
  },
  "segments": [
    { "speaker": "A", "language": "en", "text": "..." },
    { "speaker": "B", "language": "en", "text": "..." }
  ]
}
```

---

## 10. コスト見積もり

1ユーザーが1日5本（単体記事エピソード、各5分相当）を生成する場合の月次コスト試算。

| サービス | 単価 | 月次使用量（概算） | 月次コスト |
|----------|------|--------------------|------------|
| Gemini 2.5 Flash（スクリプト生成） | 入力$0.15/1M token、出力$0.60/1M token | 入力 1M token、出力 200k token | 約$0.27 |
| Gemini 2.5 Flash（レコメンド） | 同上 + Context Caching | Context Cache 50k token/日 | 約$0.05 |
| OpenAI TTS | $15/1M文字 | 約 150k 文字/月（5本×5分×1000文字） | 約$2.25 |
| Cloud Run | $0.00002400/vCPU秒 | 約 3,000 秒/月 | 約$0.07 |
| Cloud Tasks | $0.40/1M タスク | 約 1,000 タスク/月 | 約$0.00 |
| Firestore | 無料枠内 | 〜1GB以下 | $0.00 |
| Cloud Storage | $0.020/GB/月 | 約 1.5GB（150ファイル × 10MB） | 約$0.03 |
| **合計** | | | **約 $2.67/月** |

> Context Caching により Gemini のレコメンドコストは大幅削減。月5USD以内の目標は達成見込み。

---

## 11. MVP スコープ

### MVP に含む（P0）

- Feed タブ：RSS取得・スワイプ操作・パーソナライズレコメンド
- Podcast タブ：単体記事の即時オンデマンド生成（Star後1〜2分）
- 日本語イントロ＋英語掛け合い本編
- Cloud Tasks を用いた並列 TTS 処理基盤
- 難易度選択（6段階）
- 再生速度調整（8段階）
- 再生位置保存（秒単位）
- 生成完了のプッシュ通知
- Settings タブ：RSS管理・デフォルト設定
- バックエンド：Cloud Run + Cloud Tasks + Firestore + Cloud Storage

### フェーズ2（P1）

- 日次ダイジェスト（複数記事を1本にまとめて毎朝自動生成）
- オフライン再生（音声ファイルのデバイスキャッシュと同期）
- ストレージ管理（キャッシュ容量確認・削除）
- Firebase Auth によるマルチユーザー管理の本格化

#### Web フロントエンド残課題（2026-06-12 追記 — WebUI リスタイル完了時の棚卸し）

Web 版（PR #7 で新デザイン適用済み）は実装済み API 契約を正としており（ADR-002）、以下は**バックエンド API の拡張が前提**の機能:

- Podcast 生成ステータス表示・ポーリング（`PodcastResponse` への `status` / `error_message` 追加が前提）
- 再生位置のサーバー同期・端末間共有（`PATCH /podcasts/:id/position` 新設が前提。現状は localStorage のみ）
- デフォルト難易度のユーザー設定（`GET/PUT /settings` 新設 + podcast-generator の難易度参照変更が前提）
- フィードの未読管理・「未読」タブ（既読管理 API が前提）
- BFF プロキシの撤去（バックエンドへの CORS 追加が前提。ADR-001）

Web 単独で追加可能なもの:

- 記事の一括スター / 相対日時表記（「3時間前」）/ E2E テスト（Playwright）/ Web Push 通知

技術的負債（ESLint 導入等。テスト型エラー・シークバーフィルは 2026-06-12 解消済み）は `docs/tech/web-frontend-tech-debt.md` を参照。

### 将来検討（P2）

- Apple Watch 対応
- 記事全文検索
- Web クリップ（ブラウザ拡張からの Star）

---

## 12. 設計決定事項

| 項目 | 決定内容 | 理由 |
|:-----|:---------|:-----|
| 音声合成 | Gemini TTS ではなく OpenAI TTS + Google TTS | Gemini ネイティブ音声は複数話者の個別制御が難しい。OpenAI TTS で話者A/Bを並列処理することで速度・品質を両立 |
| 関連ニュース検索 | MVP では同一 RSS コーパス内のタイトル類似度比較（Embeddings の簡易比較） | 外部 Web 検索 API は費用が高い。自コーパス内で十分な関連記事が見つかるケースが多い |
| 記事本文フェッチ | RSS が全文配信しない場合は `newspaper3k`（Python）でスクレイピング | 軽量かつ主要メディアに対応済み。Cloud Run Jobs 内で動かすため依存管理も容易 |
| API 認証 | 固定 API キー（GCP Secret Manager 経由）。iOS アプリに直接ハードコードしない | 個人利用のため Firebase Auth のような複雑な認証は過剰。Secret Manager で安全にバックエンドへ注入 |
| エラー時の挙動 | TTS 並列処理で一部失敗した場合はスキップして結合し `partial_failed` でステータスを記録 | 全体を失敗にするより部分的でも完成物を提供する方がユーザー体験がよい |

---

## 13. 未解決事項

| # | 項目 | 検討が必要な内容 |
|---|------|-----------------|
| 1 | スクレイピング合法性 | 各サイトの利用規約・robots.txt の確認。個人利用の範囲内かどうか |
| 2 | OpenAI TTS の話者バリエーション | alloy / echo / fable / nova / onyx / shimmer のどれを話者A/Bに割り当てるか。音質テストが必要 |
| 3 | Push 通知実装方式 | APNs + Firebase Cloud Messaging vs Cloud Run からの直接 APNs 呼び出し |
| 4 | ユーザー識別子 | MVP では UUID を端末ローカルで生成して固定。フェーズ2以降に Firebase Auth へ移行するか |
| 5 | 関連ニュース品質 | タイトル類似度での関連記事紐付けが実用的に機能するか。本文 Embedding が必要かは実装後に評価 |
