# T02: シークバーの再生済み区間フィル（amber）

**占有ファイル（これ以外は一切編集禁止）:**
- `web/components/AudioPlayerBar.tsx`
- `web/tests/components/AudioPlayerBar.test.tsx`
- `web/app/globals.css` — ただし**末尾の T11 追加ブロック（L1365 付近の「T11 統合」コメント以降）のみ編集可**。それより前（デザイン正本 `app-ui.html` からの移植部分）は 1 文字も変更禁止（ADR-003）

**参照（読むだけ）:** `docs/adr/003-web-pure-css-design-tokens.md` / `web/app/globals.css` L1374〜1420 付近（range 装飾の現状）/ デザイン正本 `docs/design/app-ui.html` の `.progress-fill`（amber グラデーション `linear-gradient(90deg, var(--amber), #F5C842)` が目標の見た目）

## 背景

シークバーは a11y のため `input[type=range]`（`className="progress-track seek-slider"`、`web/components/AudioPlayerBar.tsx` L120〜131 付近）で実装されている。T11 でトラック + サムの装飾は追加済みだが、**再生済み区間のフィルがない**（WebKit はトラック一色、Firefox も `::-moz-range-progress` 未定義）。

## 実装方式（この方式で実装すること。代替案の検討に時間を使わない）

**CSS カスタムプロパティ + 両エンジン対応:**

1. **AudioPlayerBar.tsx**: シークバーの `input` に再生率をカスタムプロパティとしてインライン付与する
   - `--seek-fill` = `currentTime / duration * 100` に `%` を付けた文字列
   - duration が 0・未定義・NaN のときは `0%`（ゼロ除算ガード必須）
   - 値の clamp（0〜100）を行う
2. **globals.css（T11 ブロック内）**:
   - WebKit: 既存の `input[type="range"].progress-track::-webkit-slider-runnable-track` の背景を `linear-gradient(90deg, var(--amber) var(--seek-fill, 0%), <現状のトラック色> var(--seek-fill, 0%))` に変更する（カスタムプロパティは擬似要素に継承されるため、ホスト要素のインライン `--seek-fill` がここで効く）。`.volume-slider` のトラックは変更しない（セレクタを分離すること）
   - Firefox: `input[type="range"].progress-track::-moz-range-progress { background: var(--amber); height: 3px; border-radius: 2px; }` を追加
   - 既存の T11 ブロック内ルールの分離・変更は必要最小限とし、変更した場合はコメントに WHY を残す

補足: T11 ブロックはデザイン正本由来ではない（range 入力はデザインに存在しない）ため、ブロック内の変更は ADR-003 の「既存セレクタ無変更」原則に抵触しない。デザイン正本移植部分（L1363 以前）は不可侵。

## TDD 手順（Red → Green）

先に `web/tests/components/AudioPlayerBar.test.tsx` に追加（既存 25 テストは無改変でパスさせ続けること）:

1. currentTime=60・duration=240 のとき、シークバー要素の `style.getPropertyValue('--seek-fill')` が `25%`
2. duration が 0（または未ロード）のとき `--seek-fill` が `0%`（NaN% にならない）
3. currentTime が duration を超える保存値でも `100%` を超えない（clamp）
4. 音量スライダーには `--seek-fill` が付かない

注意: jsdom は CSS の描画自体を検証できない。検証対象は**インラインのカスタムプロパティ値のみ**とし、gradient の見た目アサーションを書こうとしない（書けずにループする）。

## 完了条件（期待出力つき）

| コマンド | 期待出力 |
|---|---|
| `cd web && npx vitest run tests/components/AudioPlayerBar.test.tsx` | 全テストパス（既存 25 + 新規 4 = 29 目安、fail 0） |
| `cd web && npx tsc --noEmit 2>&1 | grep 'AudioPlayerBar' | wc -l` | `0` |
| `cd web && sed -n '1,1363p' app/globals.css | git diff --no-index --quiet /dev/stdin <(git show HEAD:web/app/globals.css | sed -n '1,1363p'); echo $?` 相当の確認（デザイン正本移植部分が無変更であること。手段は `git diff app/globals.css` の差分行が T11 ブロック内のみであることの目視確認でよい） | 差分が L1365 以降のみ |

## 停止条件（処理ループ防止）

- 新規テストが 3 回の実装修正でグリーンにならない → 直前のグリーン状態に戻して停止し、失敗テストの出力全文を添えて報告
- 既存テストが 1 件でも落ちる状態を 2 回連続で作った → 変更を最小化して停止・報告
- jsdom で gradient / 擬似要素の検証を試みない（上記「注意」参照）

## 禁止事項

- globals.css のデザイン正本移植部分（L1363 以前）の変更
- range 入力の div 化・ライブラリ導入・再生ロジック（useAudioPlayer / seek 処理）の変更
- 音量スライダーへのフィル追加（スコープ外。シークバーのみ）
- ブラウザ実機確認の代替として dev サーバーを起動し続けること（実機の見た目確認は完了報告の「人間による確認推奨」に記載して委ねる）
