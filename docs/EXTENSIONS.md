# WebUI 拡張

Hermes WebUI は、セルフホストインストール向けに小さなオプトインの拡張サーフェスをサポートします。管理者が WebUI のソースツリーを編集せずに、ローカル静的アセットを配信し、same-origin の CSS や JavaScript をアプリシェルに注入できます。

> **信頼モデル — まずこれを読んでください。** 拡張は WebUI のセッション権限フルで実行されます。拡張 JS ファイルは、ログイン中ユーザーが呼べるあらゆる API を呼べます。会話履歴の読み取り、メッセージ送信、設定変更、ツールアクションのトリガーを含みます。**自分で書いた拡張、または WebUI のソースと同程度に信頼できるソースの拡張のみを有効化してください。** WebUI を完全には信頼できないユーザーと共有しているなら、拡張を有効化しないでください。`HERMES_WEBUI_EXTENSION_DIR` をユーザー書き込み可能なディレクトリに向けないでください。

これは意図的にプラグインマーケットプレイスや依存システムではありません。コア Hermes WebUI に置くべきでないローカルダッシュボード、内部ツール、ワークフロー固有のパネルのための安全な escape hatch です。

## 拡張ができること

拡張は次ができます:

- 設定された1つのローカルディレクトリから `/extensions/...` でファイルを配信
- 設定された same-origin スタイルシートを `<head>` に注入
- 設定された same-origin スクリプトを `</body>` の前に注入
- ブラウザセッションで利用可能な通常の WebUI API を呼ぶ

拡張がそれ自体ではできないこと:

- WebUI 認証のバイパス
- 設定された拡張ディレクトリ外のファイル配信
- 組み込み注入設定経由のサードパーティスクリプト/スタイルの読み込み
- 既にそれらの変更を許す既存の認証済み API を呼ぶ場合を除き、Hermes Agent の権限・モデル・メモリ・ツールの変更

## 設定

拡張はデフォルトで無効です。WebUI サーバー起動前に環境変数で設定してください。`HERMES_WEBUI_EXTENSION_DIR` は、スクリプトやスタイルシート URL が注入される前に既存ディレクトリを指す必要があります:

```bash
export HERMES_WEBUI_EXTENSION_DIR=/path/to/my-extension/static
export HERMES_WEBUI_EXTENSION_SCRIPT_URLS=/extensions/app.js
export HERMES_WEBUI_EXTENSION_STYLESHEET_URLS=/extensions/app.css
./start.sh
```

複数 URL はカンマ区切り可:

```bash
export HERMES_WEBUI_EXTENSION_SCRIPT_URLS=/extensions/runtime.js,/extensions/app.js
export HERMES_WEBUI_EXTENSION_STYLESHEET_URLS=/extensions/base.css,/extensions/theme.css
```

## URL ルール

注入されるアセット URL は意図的に制限されます:

- same-origin パスでなければならない
- `/extensions/` または `/static/` で始まらなければならない
- URL スキーム、ホスト、フラグメント、クォート、山括弧、改行、NUL バイト、バックスラッシュを含んではならない

許可される例:

```text
/extensions/app.js
/extensions/app.css
/extensions/app.js?v=1
/static/theme.css
```

拒否される例:

```text
https://example.com/app.js
//example.com/app.js
javascript:alert(1)
/api/session
/extensions/app.js#fragment
```

これらの制限は既存の Content Security Policy を保ち、拡張フックをサードパーティスクリプトローダに変えることを避けます。無効な設定 URL は注入されず無視されます。

## 静的ファイル配信

`HERMES_WEBUI_EXTENSION_DIR` が既存ディレクトリを指すとき、そのディレクトリ下のファイルが `/extensions/` 以下で利用可能になります:

```text
/path/to/my-extension/static/app.js  ->  /extensions/app.js
/path/to/my-extension/static/ui.css  ->  /extensions/ui.css
```

静的ハンドラはサンドボックス化されています:

- パストラバーサルは拒否（エンコードされたトラバーサルを含む）
- dotfile と dot ディレクトリは配信されない
- 拡張ディレクトリ外に解決されるシンボリックリンクは拒否
- 欠如または無効な拡張ディレクトリは無効として振る舞う
- 失敗はローカルファイルシステムパスを露出せず一般的な 404 を返す

## セキュリティ注記

自分が管理するディレクトリの拡張のみ有効化してください。拡張 JavaScript は WebUI オリジンで動き、ログイン中のブラウザセッションと同じ認証済み WebUI API を呼べます。

共有またはリモート公開されたインストールでは:

- `HERMES_WEBUI_PASSWORD` を有効に保つ
- 意図的に公開する場合を除きループバックにバインド
- 有効化前に拡張コードをレビュー
- 小さく監査可能な拡張ファイルを優先
- 生成された、またはユーザー書き込み可能なディレクトリを拡張ルートとして配信しない

## 拡張作成ガイダンス

拡張は WebUI アプリとページを共有するので、付加的かつ可逆であるべきです。組み込みの Chat、Tasks、Settings、セッションビューを壊さずに除去・非表示にできる、小さくスコープのよい DOM 変更を優先してください。

推奨パターン:

- 一意の ID やクラスプレフィックスで拡張固有のコンテナを作る
- 大きなアプリコンテナを置き換えるのでなく、既存ビューの隣に UI を追加
- 可能な限りイベントリスナーを拡張所有の要素にスコープ
- 組み込みナビゲーション挙動を保ち、変更したビュー状態を復元
- パネルやオーバーレイに `hidden`、`aria-*`、拡張スコープの CSS を使う
- 初期化をガードし、再読み込みや再注入でボタン・パネル・タイマー・イベントリスナーが重複しないようにする

`document.body.innerHTML`、`main.innerHTML`、その他の広い WebUI コンテナを置き換えるような破壊的変更を避けてください。それらのパターンはアプリの既存パネルを除去・隠蔽し、拡張ビューを開いた後に通常のナビゲーションが復旧できなくなり得ます。

カスタムページには、専用パネルを追加し組み込みビューと並べてトグルする方を優先してください:

```javascript
(() => {
  if (document.getElementById('my-extension-panel')) return;

  const panel = document.createElement('section');
  panel.id = 'my-extension-panel';
  panel.className = 'main-view my-extension-panel';
  panel.hidden = true;
  panel.textContent = 'My extension page';

  document.querySelector('main')?.appendChild(panel);

  function showPanel() {
    document.querySelectorAll('main > .main-view').forEach((view) => {
      view.hidden = view !== panel;
    });
  }

  // showPanel() を拡張所有のボタンやメニュー項目に配線する。
})();
```

ホスト CSS が `[hidden]` を上書きする場合、次のような拡張スコープのルールを追加:

```css
.my-extension-panel[hidden] {
  display: none !important;
}
```

## 最小例

ローカル拡張ディレクトリを作成:

```bash
mkdir -p ~/.hermes/webui-extension
cat > ~/.hermes/webui-extension/app.css <<'CSS'
.my-extension-badge {
  position: fixed;
  right: 12px;
  bottom: 12px;
  padding: 6px 10px;
  border-radius: 999px;
  background: #202236;
  color: #fff;
  font: 12px system-ui, sans-serif;
  z-index: 9999;
}
CSS
cat > ~/.hermes/webui-extension/app.js <<'JS'
(() => {
  const badge = document.createElement('div');
  badge.className = 'my-extension-badge';
  badge.textContent = 'Extension loaded';
  document.body.appendChild(badge);
})();
JS
```

拡張を有効にして WebUI を起動:

```bash
HERMES_WEBUI_EXTENSION_DIR=~/.hermes/webui-extension \
HERMES_WEBUI_EXTENSION_STYLESHEET_URLS=/extensions/app.css \
HERMES_WEBUI_EXTENSION_SCRIPT_URLS=/extensions/app.js \
./start.sh
```

WebUI を開き、バッジが現れることを確認。

---

> 本ドキュメントは hermes-webui 公式リポジトリ `docs/EXTENSIONS.md` の日本語訳です。環境変数・パス・コード・URL・識別子は原文のまま表記しています。
