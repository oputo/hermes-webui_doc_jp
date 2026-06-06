# Hermes Web UI — テーマ

Hermes Web UI は **外観** を2つの独立したピッカーに分割します:

- **Theme** — モード: `System`、`Dark`、`Light`。背景・テキスト・サーフェス・クロムの色を決める。
- **Skin** — アクセントパレット: 組み込みスキンは名前付きキーとして提供。`--accent` ファミリー（アクティブ状態、リンク、フォーカスリング、プライマリアクション）のみを決める。

各々から1つずつ選んで組み合わせるため、お気に入りのアクセントを失わずに環境に応じて見た目が適応します — 純粋な CSS で、Python の変更は不要です。

---

## 外観の切り替え

**Settings パネル:** 歯車アイコン → **Appearance** をクリック。**Theme** カードが Light/Dark/System を切り替え、**Skin** グリッドが組み込みアクセントパレットを提供。プレビューは即時 — クリックすると UI が更新されます。

**スラッシュコマンド:** コンポーザで `/theme <name>` を入力。コマンドはテーマ名（`system`、`dark`、`light`）とスキン名（`default`、`ares`、`mono`、`slate`、`poseidon`、`sisyphus`、`charizard`、`sienna`、`catppuccin`、`nous`、`geist-contrast`、`zeus`）の両方を受け付けます。一致する軸を更新し、もう一方はそのままにします。

**永続化:** どちらの選択も、ちらつきのない読み込みのため `localStorage` に保存され、`POST /api/settings` 経由でサーバー側にも保存されます（`settings.json` の `theme` と `skin` キー）。

---

## 組み込みテーマ

| Theme | 説明 |
|-------|-------------|
| **System**（デフォルト） | OS の `prefers-color-scheme` 設定に従い、ライブで更新。 |
| **Dark** | 深いダークサーフェス、長時間セッション向けの低グレア。 |
| **Light** | 明るいサーフェスに濃いテキスト、日中環境向けの高コントラスト。 |

テーマは `<html>` のクラスとして適用されます。ダークモードでは `.dark` が付き、ライトでは無し。System モードはランタイムで OS 設定を追跡します。

---

## 組み込みスキン

| Skin | 説明 |
|------|-------------|
| **Default** | オリジナルの Hermes ゴールドアクセント。暖かく控えめ。 |
| **Ares** | 燃えるような赤。高エネルギーで力強い。 |
| **Mono** | ニュートラルグレー。集中を妨げない、深い没入向け。 |
| **Slate** | スレートブルーグレー。控えめで成熟。 |
| **Poseidon** | オーシャンブルー。長時間セッション向けの落ち着きと集中。 |
| **Sisyphus** | 鮮やかな紫。うるさくならず独特。 |
| **Charizard** | 暖かいオレンジ。エネルギッシュで目に優しい。 |
| **Sienna** | 暖かい粘土と砂のアースパレット。柔らかく自然。 |
| **Catppuccin** | Mauve アクセントの Catppuccin Latte/Mocha パレット。 |
| **Nous** | スチールブルーアクセントに破線のテクニカルサーフェス。 |
| **Geist Contrast**（`geist-contrast`） | Geist 風のモノクロサーフェスに、控えめなダークモード `#FFF175` アクセント。 |
| **Zeus** | デフォルトのゴールドアクセントを保つ OLED 近似黒のダークサーフェス。ダーク重視。ライトモードではデフォルトのライトパレットにフォールバック。 |

各スキンはライト ＋ ダークのペア変種を定義し、どちらのテーマでも綺麗に読めます。スキンは `<html>` の `data-skin="<name>"` として適用されます（デフォルトスキンは属性をクリア）。

---

## カスタムスキンの作成

スキンは、ライトとダーク両方の変種でアクセント変数を上書きする小さな CSS ブロックです:

```css
/* Light variant */
:root[data-skin="my-skin"] {
  --accent:           #2E7D32;                   /* Active states, links, primary buttons */
  --accent-hover:     #1B5E20;                   /* Hover */
  --accent-bg:        rgba(46,125,50,0.08);      /* Soft tinted backgrounds */
  --accent-bg-strong: rgba(46,125,50,0.15);      /* Highlighted backgrounds */
  --accent-text:      #1B5E20;                   /* Text on accent bg */
}

/* Dark variant — usually lighter or more saturated for contrast */
:root.dark[data-skin="my-skin"] {
  --accent:           #66BB6A;
  --accent-hover:     #43A047;
  --accent-bg:        rgba(102,187,106,0.08);
  --accent-bg-strong: rgba(102,187,106,0.15);
  --accent-text:      #66BB6A;
}
```

提供方法は2通り:

1. **リポジトリ内（組み込み）:** ブロックを `static/style.css` に追加し、Settings のスキンピッカー（`static/index.html`）と `/theme` コマンドリスト（`static/commands.js`）に登録し、PR を開く。

2. **セルフホスト（fork なし）:** WebUI 拡張サーフェスを使う — `docs/EXTENSIONS.md` を参照。CSS を `HERMES_WEBUI_EXTENSION_DIR` に置き、`HERMES_WEBUI_EXTENSION_STYLESHEET_URLS` で宣言。コード変更不要。スキン属性は自分の JS から設定できます。

### Tips

- **両テーマでテスト。** Dark で映えるスキンが Light で読めないことがあります。`:root[data-skin]`（ライト）*と* `:root.dark[data-skin]`（ダーク）の両方を必ず確認。
- **`--accent-bg` 上の `--accent-text` はコントラストを取る。** strong 変種は小さなラベルやチップの背後に現れ、弱いコントラストはぼやけて見えます。
- **ロゴのグラデーションは自動的に `--accent` を使う** ため、追加作業なしでスキンに適応します。
- **サーバー変更は不要。** `settings.json` の `skin` 設定は任意の文字列を受け付けるため、CSS を読み込めばカスタムスキン名がコード変更なしで永続します。

---

## カスタムテーマの作成

完全なカスタム *テーマ*（単なるアクセント変更ではなく、全体のムードが異なるもの）は、スキンより大きな作業です。片方または両方のモードで、中核パレット変数（`--bg`、`--surface`、`--text`、`--border`、`--code-bg` など）を再定義する必要があります。コントラクトは `static/style.css` の先頭 `:root` と `:root.dark` ブロックで定義されています — そこから始めてください。

たいていの場合、実際に欲しいのはカスタム **スキン** です。既存の Light/Dark モードが合わない場合（例: 高コントラストのアクセシビリティテーマや OLED ブラック変種）のみカスタムテーマに手を伸ばしてください。

---

## フォントサイズ

**Settings → Appearance** の Theme/Skin のすぐ下: `Small`、`Default`、`Large`。`<html>` の `data-font-size` として適用され、WebUI のルートフォントサイズをスケールします。テーマやスキンと一緒に永続します。

---

## 内部の仕組み

1. **Theme:** `document.documentElement.classList.toggle('dark', isDark)` — ライトモードはクラスを除去。System モードは `matchMedia('(prefers-color-scheme: dark)')` を追跡。
2. **Skin:** `document.documentElement.dataset.skin = name`（`default` では属性を除去）。
3. **Font size:** `document.documentElement.dataset.fontSize = size`（`default` では除去）。
4. **読み込み時のフラッシュなし:** `<head>` の小さなインライン `<script>` がスタイルシートより先に `localStorage` を読み、描画前に正しい見た目を適用。
5. **サーバー同期:** 設定は `POST /api/settings` で保存され、起動時に `GET /api/settings` で再水和。

---

## スキンのコントリビュート

スキンは最も簡単な拡張ポイントです — 純粋な CSS、Python なし、JS ロジックなし。upstream に貢献するには:

1. `:root[data-skin="name"]` と `:root.dark[data-skin="name"]` のブロックを `static/style.css` に追加。
2. `static/index.html` の Settings スキンピッカーと、`static/commands.js` の `cmdTheme()` が使うスキンリストに登録。
3. Light と Dark の両テーマで、デスクトップとモバイルでテスト。
4. PR を開く — スキンは純粋な CSS 追加でバックエンド変更不要。

カスタム *テーマ*（ベースパレットの上書き）の場合は、多くのセレクタに触れるため、まず issue を開いてスコープを議論することを推奨します。

---

> 本ドキュメントは hermes-webui 公式リポジトリのルート `THEMES.md` の日本語訳です。CSS 変数・スキン名・コマンド・ファイルパス・識別子は原文のまま表記しています。
