# T06: AudioPlayerBar のリスタイル

**Wave:** 2（T01 完了後、他 Wave 2 タスクと並列実行可）
**依存:** T01
**占有ファイル:** `web/components/AudioPlayerBar.tsx`, `web/tests/components/AudioPlayerBar.test.tsx`
**参照:** `docs/design/app-ui.html` L947〜1144（CSS）, L1939〜1993（マークアップ）/ `00-overview.md` §2 D19・D20・D26

## 目的

画面下部の再生バーをデザイン（3 カラムグリッド・波形アート・amber グラデーション進捗）に置き換える。**再生ロジック（useApp / useAudioPlayerContext の利用方法）は一切変更しない**。

## 実装内容

- ルート: `<footer className="player-bar">`（既存の `aria-label="プレイヤー"` 相当のアクセシビリティ属性を維持）。`currentPodcast` が null なら**何も描画しない**（既存挙動 = D26）
- 左 `.player-track`:
  - `.player-art` > `.player-art-waveform`（バー 5 本、L1944〜1950）。一時停止中はアニメーションを止める（`isPlaying` でバーに `animation-play-state` を切り替えるインラインスタイル、または paused 用クラス付与 → クラスが CSS にない場合は申し送り）
  - `.player-info`: `.player-title`（イントロ先頭 50 文字 — 既存ロジック維持）+ `.player-subtitle`（`<DifficultyBadge>` + 生成日）
- 中央 `.player-controls`:
  - `.player-buttons`: `-15`（`.ctrl-btn`）/ 再生・一時停止（`.ctrl-btn-main`、SVG アイコンを isPlaying で切替。既存のアクセシブルネーム「再生」「一時停止」を維持）/ `+30`（`.ctrl-btn`）
  - `.player-progress`: `.progress-time`（currentTime）+ シークバー + `.progress-time end`（duration）。シークバーは**既存の `input[type=range]` を維持**し `.progress-track` 相当の見た目になるよう className 付与（見た目調整が CSS にない場合は range 入力のまま申し送り。機能 > 見た目）
- 右 `.player-extra`:
  - 速度: 既存 `<select>` を維持し `.speed-pill` クラスで装飾（D20）。選択肢・初期値ロジック不変
  - 音量: 既存 `input[type=range]`（aria-label「音量」）を維持（D19）。シャッフルボタンは実装しない

## TDD 手順（Red 観点）

既存テスト（region ロール・再生/一時停止・±シーク・シーク/音量スライダー・速度セレクタ）を**全てそのままパス**させることが最優先。追加:

1. `currentPodcast` null で何も描画されない（既存観点の確認）
2. 再生中はルートに波形要素が存在し、`isPlaying=false` で停止状態（クラス or style で検証）
3. 時間表示が `formatDuration` 形式で currentTime / duration を表示する

## 完了条件

- [x] `npm run test` 全パス
- [x] `npm run dev` で再生開始するとバーが下部に新デザインで表示され、再生/シーク/速度/音量が従来どおり動く（手動）

## 禁止事項

- `hooks/useAudioPlayer.ts`・`contexts/` の編集（ロジック不変）
- シャッフル・プレイリスト機能の追加（D19）
- range 入力の div 化など a11y を劣化させる置換
- `app/layout.tsx`・`globals.css` の編集
