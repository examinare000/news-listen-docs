# iOS デザインシステム（Editorial）

iOS アプリのビジュアルデザインの**正本**。配色・タイポグラフィ・余白・状態色のトークンと、その適用方針を定める。
Web の視覚正本 [app-ui.html](app-ui.html) に対応する iOS 版。実装の正本はコード（`ios/.../DesignSystem/`）であり、本書は
**意図・役割・トークン仕様**に徹する（実装スニペットは載せない）。

**メタデータ:**
| | |
|---|---|
| 方向性 | Editorial（雑誌・読み物的） |
| 対応モード | ライト / ダーク 両対応（適応色） |
| 書体 | 見出し=セリフ（Apple New York, `design: .serif`）／本文=SF |
| 規約 | `agent-rules/15-frontend-design.md`・[ADR-040](../adr/040-ios-editorial-design-system.md) |
| 関連 issue | [#110](../../issues/110)（iOS エディトリアルUI） |
| 最終更新 | 2026-06-30 |
| ステータス | 採用・主要画面へ適用済み |

## 1. 方向性

「白基調・システムフォント・素の `List`/`Form`」の汎用 UI を脱し、本アプリのコア価値（厳選ニュースを耳で深掘りする
読み物体験）に見合う **Editorial（雑誌風）** の一貫したデザインへ刷新する。規約 `15-frontend-design.md` の禁止
（汎用フォント Inter/Roboto/Arial/system/Space Grotesk・紋切り型配色・クッキーカッター）を避け、意図のある階層・余白・
1アクセントで品位と没入感を出す。

設計原則:
- **支配色＋1アクセント**: 温かみのある紙（paper）と墨（ink）を支配色に、朱（accent）を唯一の強調色に絞る。
- **セリフ見出し**: 見出しに Apple 純正セリフ（New York）を用い読み物的な階層を作る。本文・メタは SF。
- **トークン単一の真実**: 色・書体・余白の直値を View に散らさず、`DesignSystem/` のトークンを唯一の参照元にする。
- **アクセシビリティ非後退**: 書体はテキストスタイル基準で Dynamic Type に追従。VoiceOver ラベルは維持。

## 2. カラートークン

すべてライト/ダークの2値を持つ適応色（`UIColor(dynamicProvider:)` で実行時解決）。

### 基調色
| トークン | ライト | ダーク | 用途 |
|---|---|---|---|
| `paper` | `#FBF9F4` 温白 | `#14130F` 墨黒 | 画面全体の背景（紙） |
| `surface` | `#FFFFFF` | `#1E1C17` | カードなど一段持ち上げた面 |
| `ink` | `#1C1A17` | `#F4F0E7` | 主テキスト（墨） |
| `inkSecondary` | `#6E665B` | `#ABA496` | 副次テキスト（ソース・日時等） |
| `inkTertiary` | `#9A9286` | `#756E61` | 三次テキスト（最も控えめな補助） |
| `hairline` | `#E7E2D8` | `#322F28` | 罫線・区切り線 |
| `accent` | `#A8402E` 朱 | `#E08A6E` | 強調・関連スコア・主要操作 |
| `accentSoft` | accent 10% | accent 16% | アクセント淡面（バッジ背景） |
| `onAccent` | `#FBF9F4` | `#14130F` | アクセント面上の前景。**逆相**でライト/ダーク双方の可読性を確保 |

### セマンティック状態色
状態を表す色も直値（`.red` 等）を避けトークン化する。Editorial の紙面に馴染むよう彩度を抑える（汎用 Material 色は使わない）。

| トークン | ライト | ダーク | 用途 |
|---|---|---|---|
| `danger` | `#C0392B` | `#E57373` | エラー・破壊的操作（accent の朱と判別できる赤） |
| `success` | `#4F7A3A` | `#8BB87A` | 成功・ダウンロード完了（落ち着いた森緑） |
| `star` | `#C8902E` | `#E0B65C` | スター・強調（黄ではなく金） |

## 3. タイポグラフィ

`Font.system(_:design:)` のテキストスタイル基準で定義し、**Dynamic Type のスケールに追従**する（固定 pt を避け、
アクセシビリティを損なわない）。見出しは `design: .serif`（New York）、本文・メタは SF。

| トークン | 基準 | 用途 |
|---|---|---|
| `display` | largeTitle / serif / bold | 画面タイトル・ブランド |
| `title` | title2 / serif / semibold | セクション見出し |
| `headline` | title3 / serif / semibold | 記事タイトル等の主要見出し |
| `body` | body / SF | 本文 |
| `meta` | subheadline / SF | メタ情報（日時・長さ） |
| `caption` | caption / SF | 補助キャプション・エラー文 |
| `eyebrow` | caption / semibold / 大文字 + 字間 | ソース名などの上付きラベル |

ナビゲーションバーの大見出しも UIKit 外観でセリフへ統一する（`DSAppearance`）。SF Symbol のサイズ指定（アイコン寸法）は
タイポグラフィトークンの対象外とする。

## 4. 余白・角丸・コンポーネント

- **余白**: 4 の倍数スケール（`xs=4 / s=8 / m=12 / l=16 / xl=24 / xxl=32`）。
- **角丸**: コントロール=10 / カード=14。
- **共通コンポーネント**: `DSBadge`（ピル型ラベル）・`RelevanceBar`（関連スコアの極細バー）・`dsCard()` / `dsScreenBackground()` 修飾子。
- **Feed のレイアウト**: 角丸カードではなく**罫線区切り＋広い天地余白**の雑誌レイアウト。

## 5. 適用画面

主要画面に一貫適用する。

| 画面 | 適用内容 |
|---|---|
| Feed（一覧） | アイブロウ（ソース·日時）＋セリフ見出し＋関連スコアバー、罫線区切り・紙背景 |
| 再生（AudioPlayer） | surface カード・朱の再生ボタン・セリフのイントロ・accent シークバー |
| Podcast 一覧 | 難易度=`DSBadge`、再生中アイコン=朱、状態バッジ（生成中=accent / 失敗=danger / DL済=success） |
| 設定・アカウント | 紙背景・本文/見出しのトークン化・エラー=danger |
| ログイン | セリフのブランドヘッダ・紙背景・エラー=danger |
| オンボーディング・ユーザー管理 | 紙背景・トークン化 |

## 6. アクセシビリティ

- **Dynamic Type**: 全書体トークンがテキストスタイル基準のため拡大に追従。
- **VoiceOver**: 既存のラベル/ヒント/値を維持（刷新で非後退）。詳細は [ADR-034](../adr/034-ios-accessibility-voiceover-dynamic-type.md)。
- **コントラスト**: `onAccent` を適応色にし、明色アクセント（ダーク時）上でも前景の可読性を確保する。

## 7. 実装の所在

実装の正本はコード。トークン・コンポーネントは `ios/NewsListenApp/NewsListenApp/DesignSystem/` 配下に集約する
（`DSColor` / `DSFont` / `DSLayout` / `DSAppearance` / `Components/`）。View は直値を持たず、これらトークンのみを参照する。

## 関連
- [ADR-040](../adr/040-ios-editorial-design-system.md): iOS Editorial デザインシステムの採用決定
- [ios-design.md](ios-design.md): iOS 設計（SwiftUI/MVVM・画面・API）
- `agent-rules/15-frontend-design.md`: フロントエンドデザイン規約（リポジトリルート）
