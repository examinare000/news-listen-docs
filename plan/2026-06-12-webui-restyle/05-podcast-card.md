# T05: PodcastCard のリスタイル + 再生中強調

**Wave:** 2（T01 完了後、他 Wave 2 タスクと並列実行可）
**依存:** T01
**占有ファイル:** `web/components/PodcastCard.tsx`, `web/tests/components/PodcastCard.test.tsx`
**参照:** `docs/design/app-ui.html` L547〜694（CSS）, L1632〜1743（マークアップ実例）/ `00-overview.md` §2 D11・D24

## 目的

Podcast カードをデザインのグリッドカード（バッジ・丸型再生ボタン・2 行クランプのイントロ・再生中パルス枠）に置き換える。

## 実装内容

### props 変更（後方互換の追加のみ）

`{ podcast, onPlay, savedPosition?, playing?: boolean }` — `playing` を**省略可能**で追加（デフォルト false）。再生中判定（`currentPodcast?.id === podcast.id`）は親ページの責務（T08）。WHY: カードを Context 非依存の純粋コンポーネントに保ちテストを軽くする（ArticleCard と同方針）。

### マークアップ（L1695〜1717 の「完了エピソード」を基本形に）

- ルート: `<div className="podcast-card">`、`playing` 時は `podcast-card playing`
- `.podcast-card-top`:
  - `.podcast-badges`: `<DifficultyBadge>`（T03 で `.badge` 化される。本タスクでは現行のまま使用してよい）+ `type === 'digest'` のとき `DIGEST` タグ（L1741 のスタイルは申し送りに依存させず、`.badge` クラスで代替してよい）
  - **ステータスバッジ（生成中/完了/失敗）は実装しない**（D11）
  - `.podcast-play-btn`: 丸型再生ボタン（`<button>` 維持、既存のアクセシブルネーム「再生」維持）。クリックで `onPlay(podcast)`、カードのリンク遷移に伝播させない（`stopPropagation` / リンク外配置）
- `.podcast-intro`: `japanese_intro_text`（CSS 側で 2 行クランプ。既存の 80 文字 slice は撤去してよい — クランプが代替）
- `.podcast-footer`:
  - `.podcast-meta`: 時計アイコン + `formatDuration(duration_seconds)` / `formatDate(created_at)`
  - `savedPosition` があれば「続きから {formatDuration(savedPosition)}」（既存挙動維持。`.podcast-meta` 内に配置）
  - `playing` 時は `.waveform-mini`（バー 4 本）+「再生中」ラベル（L1654〜1661）
- 詳細ページへのリンク（`/podcast/{id}`）は既存どおり維持（イントロ部を `<Link>` で包む等、再生ボタンと干渉しない構造にする）

## TDD 手順（Red 観点）

既存テストを全てパスさせた上で追加:

1. `playing=true` でルートに `playing` クラスが付き、「再生中」テキストが表示される
2. `playing` 未指定（デフォルト）では `playing` クラスも「再生中」も出ない
3. `type='digest'` で「DIGEST」が表示され、`type='single'` では出ない
4. 再生ボタンクリックで `onPlay` が呼ばれ、詳細ページへの遷移リンクと独立している（既存観点の維持）

## 完了条件

- [x] `npm run test` 全パス
- [x] `npm run dev` の /podcast でカードがデザイン外観になる（再生中強調の画面確認は T08 結合後＝T11 で実施）

## 禁止事項

- `app/podcast/page.tsx`・`[id]/page.tsx` の編集（T08 の占有）
- AppContext / AudioPlayerContext の参照追加（再生中判定は props 経由）
- ステータス・ポーリング関連の実装（D11）
- `globals.css` の編集
