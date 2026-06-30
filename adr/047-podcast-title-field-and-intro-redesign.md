# ADR-047: Podcast に `title` フィールド新設と冒頭挨拶の廃止

**ステータス:** 採用済み
**決定日:** 2026-06-30
**対象読者:** バックエンド / Web / iOS 開発者
**関連:** [ADR-006](006-cross-user-podcast-cache.md)（Podcast 生成キャッシュ）・[ADR-023](023-tts-partial-failure-skip-join.md)（TTS 一部失敗）・GitHub issue #65, #45, #41

## 背景

従来のポッドキャスト台本は「週刊ニュース番組」風の冒頭挨拶（「今週のニュース…」「こんにちは。週刊…」等）で始まり、記事内容の判別がユーザーにとって難しかった。クライアント側でもポッドキャストの内容を一覧表示するときに、`japanese_intro_text` のみを表示していたが、複数エピソード間での識別性が低かった。

また、API レスポンスに明示的な「1セン テンス要約」フィールドを持たせることで、クライアント UI の言語固有ロジック（text parsing や display fallback）を統一でき、将来の多言語化にも対応しやすくなる。

## 検討した選択肢

| 選択肢 | メリット | デメリット |
|---|---|---|
| （現状）`japanese_intro_text` のみ利用 | 変更なし | 識別性低・多言語対応困難・UI ロジックが分散 |
| `japanese_intro_text` を短縮（10文字以内に強制） | 実装小 | 要約品質低・クライアント側で不足時に補完必須 |
| **新フィールド `title`（1センテンス要約）を Gemini 出力から抽出・後方互換 default 付き**（採用） | 明示的・end-to-end 仕様・クライアント fallback で前向き互換性・多言語拡張対応 | スクリプト生成指示変更・抽出パーサ実装・グレースフル縮退設計が必須 |

## 決定

### 1. スクリプト生成の変更

- **冒頭挨拶の廃止**: `jobs/podcast_generator/script_generator.py` の `_SCRIPT_PROMPT` / `_DIGEST_SCRIPT_PROMPT` で、
  「週刊ニュース番組」風の固定挨拶（「今週のニュース…」「こんにちは、週刊…」）を廃止。
  代わりに「こんにちは。今回取り上げるニュースは……」形式の簡潔な冒頭に統一し、記事内容を直接述べる。
  
- **TITLE セクション追加**: Gemini 出力に `===TITLE===\n{1sentence}\n===END TITLE===` マーカーを埋め込み、
  1 セン テンス日本語要約を機械抽出可能にする。

### 2. データモデルとバックエンド実装

- **Podcast/ PodcastCache データモデル**: `title: string | null` フィールドを新設（後方互換 default: null）。
  
- **`_parse_script` パーサの強化**: Gemini 出力を以下の順序で partition 化：
  ```
  [TITLE セクション（Optional）] → [JP イントロ] → [EN 本編]
  ```
  TITLE マーカー（`===TITLE===…===END TITLE===`）が存在すれば抽出して `title` に格納。
  マーカー欠落・順序違反・解析エラー → `title=null` で続行（グレースフル縮退）。
  JP イントロ / EN 本編の抽出は既存ロジック（失敗時 `partial_failed` または `failed`）に依存。

- **Firestore 保存**:
  - `podcasts` コレクション: `title` フィールド追加。既存ドキュメント（null ）と新規（Gemini 抽出値）は互換。
  - `podcastCache` コレクション: キャッシュ共有キーには `title` を含めない（キー：`article_id__difficulty__language`）。
    キャッシュヒット時のクライアント側表示は per-user Podcast の `title` を使用。

### 3. API 契約の拡張

- **レスポンス**: `PodcastResponse` に `title?: string | null` を追加。
  OpenAPI スキーマは Optional で、既存クライアントのパース互換性を保証。
  
- **後方互換性**: 既存クライアントが `title` を無視しても動作不変。

### 4. クライアント側の消費（実装は別 PR / ADR）

- **Web**: `title` が存在すれば一覧表示・詳細表示で優先利用。欠落/null 時は従来の `japanese_intro_text` にフォールバック。

- **iOS**: SwiftUI `PodcastResponse` に `title` フィールド追加。UIView で表示時に、
  `title ?? japaneseIntroText ?? "ニュースポッドキャスト"`（最終デフォルト）で順序立てたフォールバック。

## 結果

1. **ユーザー体験**: ポッドキャスト一覧でエピソード内容が一目瞭然（記事タイトル風の要約）。冒頭挨拶が簡潔になり、記事本編へのリード時間が短縮。

2. **実装の堅牢性**: TITLE セクション抽出の失敗は `title=null` で自動処理し、ジョブを失敗させない（end-to-end のグレースフル縮退）。

3. **API 安定性**: 既存クライアント・Firestore schema 互換。データベースマイグレーション不要。

4. **将来性**: `title` が明示的フィールドになることで、多言語化時に `title_en`・`title_zh` 等への拡張が容易。

---

## 実装の詳細（設計書記述）

詳細は [backend-design.md §3 バッチ処理](../design/backend-design.md#3-バッチジョブ処理)（podcast-generator-job）・[§9 Firestore データモデル - podcasts コレクション](../design/backend-design.md#podcasts-コレクション) を参照。
