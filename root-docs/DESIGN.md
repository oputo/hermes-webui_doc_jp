---
version: alpha
name: Hermes Calm Console
description: "A restrained agent control surface: conversational content first, tool traces as quiet metadata, minimal chrome."
colors:
  primary: "#EAE0D5"
  secondary: "#C6AC8F"
  tertiary: "#C6AC8F"
  neutral: "#0A0908"
  surface: "#22333B"
  surfaceSubtle: "#11100E"
  borderSubtle: "#3B4A50"
  ink: "#0A0908"
  success: "#86C08B"
  warning: "#E0B15D"
  error: "#F87171"
typography:
  body-md:
    fontFamily: "Georgia, Times New Roman, serif"
    fontSize: 15px
    fontWeight: 400
    lineHeight: 1.68
  body-sm:
    fontFamily: "-apple-system, BlinkMacSystemFont, Segoe UI, Inter, system-ui, sans-serif"
    fontSize: 12px
    fontWeight: 400
    lineHeight: 1.45
  user-message:
    fontFamily: "-apple-system, BlinkMacSystemFont, Segoe UI, Inter, system-ui, sans-serif"
    fontSize: 14px
    fontWeight: 400
    lineHeight: 1.55
  mono-xs:
    fontFamily: "SF Mono, ui-monospace, monospace"
    fontSize: 11px
    fontWeight: 500
    lineHeight: 1.55
rounded:
  sm: 4px
  md: 8px
  lg: 12px
  pill: 999px
spacing:
  xs: 4px
  sm: 8px
  md: 12px
  lg: 16px
components:
  app-shell:
    backgroundColor: "{colors.neutral}"
    textColor: "{colors.primary}"
    rounded: "{rounded.sm}"
    padding: 16px
  panel:
    backgroundColor: "{colors.surface}"
    textColor: "{colors.primary}"
    rounded: "{rounded.lg}"
    padding: 16px
  border-line:
    backgroundColor: "{colors.borderSubtle}"
    textColor: "{colors.primary}"
    rounded: "{rounded.sm}"
    padding: 4px
  state-success:
    backgroundColor: "{colors.success}"
    textColor: "{colors.ink}"
    rounded: "{rounded.sm}"
    padding: 4px
  state-warning:
    backgroundColor: "{colors.warning}"
    textColor: "{colors.ink}"
    rounded: "{rounded.sm}"
    padding: 4px
  state-error:
    backgroundColor: "{colors.error}"
    textColor: "{colors.ink}"
    rounded: "{rounded.sm}"
    padding: 4px
  tool-call-group:
    backgroundColor: "{colors.neutral}"
    textColor: "{colors.secondary}"
    rounded: "{rounded.md}"
    padding: 4px
  tool-card:
    backgroundColor: "{colors.surfaceSubtle}"
    textColor: "{colors.secondary}"
    rounded: "{rounded.md}"
    padding: 8px
  user-message:
    backgroundColor: "{colors.tertiary}"
    textColor: "{colors.ink}"
    rounded: "{rounded.lg}"
    padding: 12px
---

<!-- 注: 上記の YAML フロントマターはデザイントークン定義のため原文のまま保持しています。以下の本文が日本語訳です。 -->

## 概要

Hermes WebUI は、カラフルなカードを寄せ集めたデモページではなく、静かな開発者コンソールのように感じられるべきです。主役は会話です。ツール呼び出し、思考トレース、コンテキスト圧縮記録、トークン使用量、ランタイムステータスは有用ですが、トランスクリプトのメタデータであり、ユーザーとアシスタントの文章より視覚的優先度を下げるべきです。

目指す方向性は、Linear/Vercel のような精密さに、少しの Claude 風の会話的な温かさを加えたものです。静かなサーフェス、明確な余白、控えめなアクセント使用、そしてデバッグ詳細の段階的開示。

## 色

- **Primary (#EAE0D5):** ダークサーフェス上の主要テキスト。暖かいパーチメントは、明るい白のターミナルテキストではなく、読みやすく落ち着いて感じられるべき。
- **Secondary/Tertiary (#C6AC8F):** メタデータと控えめなアクセント。アクティブ状態、フォーカス、ユーザー吹き出し、静かな強調に控えめに使う。
- **Neutral (#0A0908):** アプリ背景とインク。以前の navy/gold テーマに戻ることなく WebUI に奥行きを与える。
- **Surface (#22333B):** パネル、サイドバー、より強い対話サーフェス。会話を主役に保ちつつ構造を担う。
- **Light surfaces (#EAE0D5 / #F4EEE7):** ライトモードはパレットのパーチメントを地として使い、パネルにはわずかに持ち上げた派生サーフェスを使う。
- **Semantic colors:** success/warning/error/info は状態色のみで、装飾的なパレット選択ではない。

## タイポグラフィ

Claude 風のスプリットタイポグラフィを使う。アシスタントの文章は編集的なセリフ stack（Anthropic Serif の代替として利用可能な Georgia）を、ユーザー吹き出しと機能的 UI はくっきりしたサンス stack を保つ。これにより、コントロールを書物的に感じさせずに、ボットの声を静かで読みやすく保てる。monospace はコード、ファイルパス、コマンド、ツール名、コンパクトなメタデータのみに使う。実際にログである場合を除き、カード全体をターミナル出力のように感じさせない。

スケールはタイトに保つ: 11px メタデータ、12px ラベル、14px 本文、16〜18px 見出し。実際のレイアウト制約がない限り、10px/10.5px/12.5px の一回限りを増やさない。

## レイアウト

会話のリズム:

1. ユーザーメッセージ — 右寄せ、コンパクトな吹き出し。
2. アシスタントコンテンツ — 左寄せ、文章優先、重い吹き出しなし。
3. ツール/思考/コンテキストのトレース — アシスタントターン内の静かな開示行。
4. 生ログ/詳細 — 明示的に展開するまで隠す。

メタデータは読みの流れを断ち切るべきではない。10個のツールを使ったターンは、10個のコンテンツカードではなく、1つのコンパクトな `Used 10 tools` 開示を伴う1つのアシスタントターンとして読めるべき。

## エレベーションと奥行き

トランスクリプトではほぼ影を使わない。影はポップオーバー、ドロップダウン、モーダルダイアログ、フローティングコントロールに留保。チャット内のカードは、控えめなボーダーか控えめな tint のどちらかを使い、両方を強くは使わない。

## 形

- 行/リスト項目: `4〜8px` の radius。
- カード/パネル: `8〜12px` の radius。
- Pill: 真のチップ/バッジのみ `999px` を使う。
- 入れ子の角丸長方形の積み重ねを避ける。カードが別のカードを含むなら、おそらく一方は不要。

## コンポーネント

### ツール/思考のアクティビティグループ

落ち着いた履歴中とライブ実行中はデフォルトで折りたたみ（ユーザーがそのアクティビティ行を以前に明示的に開いていない限り）。開閉の開示状態をチャットごと・ターンごとに永続化し、チャットを離れて戻ってもユーザーが残したモードを保つ。要約行は内部情報に1つの開示を使い、意図的に簡潔に保つ（例: `Activity: 4 tools`）。常に存在する思考エリアを繰り返したり、個々のツール名を列挙したり、2つ目の末尾カウントバッジを追加したりすべきではない。展開すると思考と個々のツールカードを一緒に表示。注意を要するエラーや承認状態がない限り、思考とツールは別々のトランスクリプト行を作るべきではない。

### ツールカード

ツールカードはデバッグイベント行であり、チャットメッセージではない。アイコン、名前、短いターゲット/プレビュー、ステータスを表示。引数と結果スニペットは展開の背後に。結果スニペットは切り詰め、完全なログは「show more」の背後に。

### 思考/コンテキストカード

ツール呼び出しメタデータと同じ視覚ファミリー。アシスタントの文章より静かであるべきで、ユーザーが展開しない限り明るい tint のフルカードを使わない。

### コンポーザ

コンポーザはコマンドサーフェス。読みやすく集中させて保つ: 控えめな radius、控えめなボーダー、非アクティブチップは透明、大げさなホバースケーリングなし。

## Do's and Don'ts

Do:

- ノイジーなエージェント内部情報をデフォルトで折りたたむ。
- アクセント色は一度に1つ。
- ニュートラルなボーダーと控えめなサーフェスを優先。
- デバッグトレースを、視覚的に支配的にせずアクセス可能・検査可能にする。
- 将来の視覚回帰テストのために安定したクラス/data フックを追加。

Don't:

- すべてのツール呼び出しを第一級のチャットカードとして描画する。
- 同じビューポートで gold、cyan、purple、orange、red、green を装飾色として混ぜる。
- トークンが存在するのに新しいハードコードの radius/色の値を追加する。
- 通常コントロールに影、グラデーション、ホバートランスフォームを使う。
- 重要なエラーや承認状態を隠す。それらはアクションを要するため目立ってよい。

---

> 本ドキュメントは hermes-webui 公式リポジトリのルート `DESIGN.md` の日本語訳です。冒頭の YAML フロントマター（デザイントークン）は構造化データのため原文のまま保持し、本文を翻訳しています。色コード・変数・数値・識別子は原文のまま表記しています。
