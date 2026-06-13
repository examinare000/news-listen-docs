# ADR-011: Podcast 生成状態の表示はバックエンド変更を前提とし Phase 1 では対象外とする

**ステータス:** 検討中
**決定日:** 2026-06-14
**対象読者:** iOS 開発者・バックエンド開発者
**関連:** [ADR-005](005-job-trigger-on-action.md)・[ADR-006](006-cross-user-podcast-cache.md)・[要件 §1.1](../plan/2026-06-14-ios-app-requirements.md)

## 背景

Star 操作で `podcast-generator` ジョブが非同期起動し、生成完了後に per-user Podcast が作成される（[ADR-005](005-job-trigger-on-action.md)・[ADR-006](006-cross-user-podcast-cache.md)）。先行設計案は「生成中（processing）エピソードを `status` で判定し 5 秒ごとにポーリング表示」する想定だった。しかし、

- `Podcast` モデルには `status`（processing/completed/failed/partial_failed）が存在するが、**現行 API レスポンスには含まれていない**。
- 専用の生成状態取得エンドポイントも存在しない。
- 現行 Web は生成状態を表示しておらず、完成したエピソードが `GET /podcasts` に現れることで結果を知る方式。

つまり「生成中」表示は Web パリティの範囲外であり、実現にはバックエンド変更が必要。

## 検討した選択肢

| 選択肢 | メリット | デメリット |
|---|---|---|
| **Phase 1 は生成状態表示なし（Web と同等）** | バックエンド変更不要・スコープ最小。再取得で完成エピソードが現れる | 生成完了までユーザーが結果を確認できない（Web と同じ UX） |
| `PodcastResponse` に `status`/`error_message` を追加しポーリング表示 | 「生成中/失敗」を可視化でき UX 向上 | バックエンド変更（API スキーマ拡張）が必要。別ブランチ・別意思決定 |
| APNs で生成完了通知（[Phase 2 候補]） | プッシュで即時通知・ポーリング不要 | バックエンドの通知基盤・デバイストークン管理が必要。大きな追加開発 |

## 決定

Phase 1（Web パリティ）では生成状態の表示・ポーリングを**実装しない**。Podcast 一覧/詳細の取得（pull-to-refresh 含む）で完成エピソードを表示する Web と同等の挙動とする。

生成状態の可視化は Phase 2 課題として保留する。実現する場合は (a) `PodcastResponse` への `status`/`error_message` 追加 + クライアントポーリング、または (b) APNs 通知のいずれかを別途 ADR で確定する。

## 理由

`status` 表示は現行 API では実現できずバックエンド変更を要する。Phase 1 の目的は Web パリティであり、Web も状態表示を持たないため、スコープを限定して見送るのが整合的。

## 結果（影響）

- iOS の `Podcast` モデルは現行レスポンスに合わせ `status` を持たせない（実在しないフィールドを定義しない）。
- ステータス可視化を採用する際は、バックエンドの API スキーマ変更とセットで本 ADR を「採用済み」へ更新、または上書き ADR を作成する。
- Phase 2 候補（生成完了の APNs 通知）は [PRD](../prd/2026-05-31-news-listen.md) に追補する。
</content>
