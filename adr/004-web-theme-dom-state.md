# ADR-004: テーマ切替は html[data-theme] を単一の真実とし React 状態に保持しない

**ステータス:** 採用済み
**決定日:** 2026-06-12
**対象読者:** Web フロントエンド開発者

## 背景

WebUI リスタイルでダーク/ライトテーマ切替を導入した。テーマ状態をどこに保持し、初期値をどう適用するかの決定が必要だった。React アプリでは Context にテーマを持つ実装が一般的だが、本アプリの全スタイルは CSS 変数駆動であり、テーマ切替に React の再レンダリングは本質的に不要。

## 検討した選択肢

| 選択肢 | メリット | デメリット |
|---|---|---|
| React Context（ThemeProvider）にテーマ state を保持 | React の流儀に沿う | テーマ切替で全コンポーネント再レンダリング。Provider 追加。SSR 初期値と localStorage の不整合処理が複雑 |
| **DOM の `html[data-theme]` 属性を単一の真実とする** | CSS 変数だけで切替が完結し再レンダリング不要。初期適用を React 起動前に行える | React の外に状態がある（テストでは DOM 属性を直接検証） |

## 決定

- テーマは `document.documentElement.dataset.theme`（`dark` / `light`）が唯一の状態。CSS は `html[data-theme="light"]` セレクタでトークンを上書きする
- 初期値は `layout.tsx` の `<head>` 内**同期インラインスクリプト**で適用: localStorage の `theme` キー → なければ `prefers-color-scheme`。`<html suppressHydrationWarning>` を付与
- localStorage には**生文字列**（`"dark"` / `"light"`、JSON 引用符なし）で保存する。トグルコンポーネントは `useLocalStorage` フック（JSON.stringify する）を使わず `localStorage.setItem` を直接呼ぶ
- トグル UI は `role="switch"` + `aria-checked` で状態を支援技術へ公開。そのための鏡像 state はトグル自身に閉じる（グローバル再レンダリングは発生しない）

## 理由

- インラインスクリプトによる React 起動前の適用が FOUC（テーマちらつき）を防ぐ唯一の方法
- 生文字列保存は、初期化スクリプト（JSON.parse を持たない素の `localStorage.getItem`）との互換のため
- スタイルが CSS 変数駆動（ADR-003）である以上、React 再レンダリングを伴う Context 管理はコストだけが残る

## 結果（影響）

- テーマを参照したい新規コンポーネントは React state ではなく `dataset.theme` を読む（必要ならローカルに鏡像を持ち `aria-*` にのみ使う）
- `theme` キーの値形式（生文字列）を変更する場合は、初期化スクリプトとトグルの両方を同時に変更すること（テストで一致を担保済み）
