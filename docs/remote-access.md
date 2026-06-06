# リモートアクセス

セルフホストの Hermes WebUI に、別のマシンやスマホから到達する方法。

## リモートマシンからのアクセス

サーバーはデフォルトで `127.0.0.1`（ループバックのみ）にバインドします。Hermes を VPS やリモートサーバーで動かしている場合は、ローカルマシンから SSH トンネルを使ってください:

```bash
ssh -N -L <local-port>:127.0.0.1:<remote-port> <user>@<server-host>
```

例:

```bash
ssh -N -L 8787:127.0.0.1:8787 user@your.server.com
```

その後、ローカルブラウザで `http://localhost:8787` を開きます。

`start.sh` は、SSH 経由で実行していることを検出すると、このコマンドを自動的に表示します。

---

## Tailscale でスマホからアクセスする

[Tailscale](https://tailscale.com) は WireGuard 上に構築されたゼロ設定のメッシュ VPN です。サーバーとスマホの両方にインストールすると、両者が同じプライベートネットワークに参加します。ポートフォワーディングも、SSH トンネルも、公開露出も不要です。

Hermes Web UI はモバイル最適化レイアウト（ハンバーガーサイドバー、ドロワー内のサイドバー上部タブ、タッチしやすいコントロール）で完全にレスポンシブなので、スマホから日常使いのエージェントインターフェースとしてよく機能します。

**セットアップ:**

1. サーバーと iPhone/Android の両方に [Tailscale](https://tailscale.com/download) をインストールします。
2. パスワード認証を有効にして、すべてのインターフェースで待ち受ける WebUI を起動します:

```bash
HERMES_WEBUI_HOST=0.0.0.0 HERMES_WEBUI_PASSWORD=your-secret ./start.sh
```

3. スマホのブラウザで `http://<server-tailscale-ip>:8787` を開きます（サーバーの Tailscale IP は Tailscale アプリ、またはサーバー上で `tailscale ip -4` で確認できます）。

これだけです。トラフィックは WireGuard によりエンドツーエンドで暗号化され、パスワード認証がアプリケーションレベルで UI を保護します。ホーム画面に追加すれば、アプリのような体験になります。

### コミュニティ報告: AVF 経由の ARM64 Android

[#2364](https://github.com/nesquena/hermes-webui/issues/2364) のコミュニティ報告では、Hermes Agent ＋ WebUI が、Android Virtualization Framework（AVF）経由の Debian 12 VM 内で、ミドルレンジの ARM64 Android スマホ上で動作したことが記録されています。報告された構成は、Xiaomi Redmi Note 13 Pro 4G、VM に割り当てた 3.8 GiB RAM、8 個の可視 CPU コア、Android 上の Chrome で `localhost:8787`、クラウドホスト型の推論でした。

これは公式のサポート基準やプロバイダー/モデルのベンチマークではありませんが、モバイル ARM64 実験の有用な互換性シグナルです。WebUI は Chrome で滑らかに描画され、ARM64 Debian がエージェントスタックで動作し、ローカルの総フットプリントは約 1.7 GB でした。報告からの実務上の注意点: 依存関係をソースからコンパイルすると初回インストールに時間がかかること、アプリ切り替え時に Android のブラウザタブが再読み込みされる場合があること、長時間セッションではターミナルや VM ホストのバッテリー最適化を無効化する必要がある場合があること。

> **Tip:** Docker を使う場合は、`docker-compose.yml` の environment で `HERMES_WEBUI_HOST=0.0.0.0`（既にデフォルト）を設定し、`HERMES_WEBUI_PASSWORD` を設定してください。

---

> 本ドキュメントは hermes-webui 公式リポジトリ `docs/remote-access.md` の日本語訳です。コマンド・環境変数は原文のまま表記しています。
