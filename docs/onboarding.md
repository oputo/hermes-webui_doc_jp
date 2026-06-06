# 初回起動オンボーディングガイド

このガイドは、Hermes WebUI を初めて起動したときに何が起こるか、どのセットアップ経路を選ぶか、そしてウィザードが完了できないときの復旧方法を説明します。

AI アシスタントがインストール、再インストール、bootstrap、プロバイダー設定、初回起動サポートを手伝う場合は、コマンドを実行したりログを調べたりする前に [`docs/onboarding-agent-checklist.md`](onboarding-agent-checklist.md) を読んでください。

短く言うと: bootstrap を実行し、WebUI を開き、プロバイダーを選び、ワークスペースを選び、任意でパスワードを設定し、チャットを開始します。Docker からローカルモデルサーバーを使う場合は、後述の Base URL セクションに特に注意してください。

## 始める前に

Hermes WebUI はブラウザインターフェースに過ぎません。実際のエージェントランタイム、メモリ、スキル、設定、cron ジョブ、プロバイダー認証情報は Hermes Agent に属します。

bootstrap は Linux、macOS、WSL2 に対応します。ネイティブ Windows はまだ bootstrap で非対応です。コミュニティによるネイティブ Windows セットアップは [#1952](https://github.com/nesquena/hermes-webui/issues/1952) で追跡されています。内容:

- [Native Windows guide](https://github.com/markwang2658/hermes-windows-native-guide)
- [Native Windows setup scripts](https://github.com/markwang2658/hermes-windows-native)

今日サポートされた経路を使いたい Windows ユーザーは、WSL2 を使い、[Windows / WSL 自動起動](wsl-autostart.md) を参照してください。

## インストール経路の選択

| 経路 | 使う場面 | 注記 |
|---|---|---|
| ローカル bootstrap | Linux・macOS・WSL2 上で WebUI を直接動かす | 個人サーバー、Mac mini、VPS、ホームラボホストに最適。 |
| Docker シングルコンテナ | 最も簡単なコンテナ構成が欲しい | 推奨される最初の Docker 経路。WebUI がエージェントをインプロセスで実行する。 |
| Docker 2コンテナ | エージェントの gateway を既に別途動かしている | より隔離されるが、WebUI から起動したツールは WebUI コンテナ内で動く。 |
| Docker 3コンテナ | エージェント gateway ＋ダッシュボード＋ WebUI が欲しい | 2コンテナと同じ注意点に加え、ダッシュボードサービスが加わる。 |
| ネイティブ Windows コミュニティ経路 | 非対応のネイティブ Windows を意図的に試す | 現状コミュニティ保守であり、公式 bootstrap 経路ではない。 |

Docker インストールが分かりにくくなったら、シングルコンテナ構成からやり直してください。UID/GID、ソースボリューム、ツール配置に関する多くの落とし穴を避けられます。完全なコンテナリファレンスは [Docker セットアップガイド](docker.md) を参照してください。

## オンボーディングを安全に再実行する

ウィザードをもう一度見るためだけに `~/.hermes` を削除しないでください。そのディレクトリには実運用の Hermes 設定、認証情報、メモリ、スキル、プロファイル、セッション、cron 状態が入っていることがあります。

クリーンなローカル試用には、隔離した Hermes ホームと WebUI 状態ディレクトリを使ってください:

```bash
mkdir -p ~/hermes-onboarding-test
HERMES_HOME=~/hermes-onboarding-test/.hermes \
HERMES_WEBUI_STATE_DIR=~/hermes-onboarding-test/webui \
HERMES_WEBUI_PORT=8789 \
python3 bootstrap.py
```

その後 `http://127.0.0.1:8789` を開きます。

アシスタント主導の試行については、[`docs/onboarding-agent-checklist.md`](onboarding-agent-checklist.md) の安全ルール、エビデンスコマンド、合否基準に従ってください。

リポジトリに `.env` ファイルがある場合、bootstrap はそれを読み込むことを忘れないでください。上記の隔離コマンドを使う前に、そこにある `HERMES_HOME`、`HERMES_WEBUI_STATE_DIR`、`HERMES_WEBUI_PORT` のエントリを削除または調整してください。

マネージドホスティングや完全に事前設定されたイメージでは、`HERMES_WEBUI_SKIP_ONBOARDING=1` を設定してウィザードをバイパスできます。

## ウィザードがチェックする内容

最初の画面は、WebUI が認識できるランタイム状態を報告します:

- Hermes Agent のインポート可否: WebUI が `AIAgent` をインポートして実行できるか。
- プロバイダー状態: `config.yaml` と認証情報の状態がチャットリクエストに足りているか。
- パスワード状態: WebUI のパスワード保護が有効か。
- 設定パス: このプロファイルで有効な `config.yaml` と `.env` の場所。

エージェントチェックが失敗したら、[トラブルシューティング](troubleshooting.md)、特に `AIAgent not available` セクションを使ってください。プロバイダー設定が未完了なら、ウィザードを進めるか、WebUI を動かすのと同じマシン環境で `hermes model` を実行してください。

## プロバイダーの選択

セットアップステップは、通常必要となる情報量でプロバイダーをグループ化します。

| グループ | 例 | 通常入力する内容 |
|---|---|---|
| Easy start | OpenRouter, Anthropic, OpenAI | API キーとモデル。 |
| Open / self-hosted | Ollama, LM Studio, カスタム OpenAI 互換 | Base URL、モデル、任意の API キー。 |
| Specialized | Gemini, DeepSeek, Xiaomi MiMo, Z.AI / GLM, NVIDIA NIM, Mistral, xAI | プロバイダー API キーとデフォルトモデル。 |

API キー型プロバイダーの場合、ウィザードはキーを有効な Hermes `.env` ファイルへ書き込み、デフォルトのモデル/プロバイダーを `config.yaml` へ書き込みます。

ローカルプロバイダーの場合、サーバーがキー不要なら API キー欄は空でかまいません。ほとんどの LM Studio、Ollama、vLLM、llama-server、TabbyAPI のインストールはこの方式です。**Test connection** を使って Base URL を検証し、続行前にモデル一覧を取得してください。

Nous Portal や GitHub Copilot のような高度なプロバイダーフローは、依然としてターミナル優先です。OpenAI Codex と Anthropic Claude Code の OAuth は、Hermes 設定が対応プロバイダーを選択しているとき、オンボーディングフロー内で開始できます。ウィザードが `hermes model` へ戻すよう指示する場合は、まずその CLI フローを使い、その後 WebUI を再読み込みしてください。

## ローカルモデルサーバーの Base URL ルール

セルフホストプロバイダーの場合、Base URL は OpenAI 互換 API のルートを指す必要があります。よくある例:

| サーバー | 典型的な Base URL |
|---|---|
| 同一の非 Docker ホスト上の LM Studio | `http://127.0.0.1:1234/v1` |
| 同一の非 Docker ホスト上の Ollama | `http://127.0.0.1:11434/v1` |
| Docker Desktop から見た LM Studio | `http://host.docker.internal:1234/v1` |
| Docker Desktop から見た Ollama | `http://host.docker.internal:11434/v1` |
| Linux Docker Engine から見たローカルサーバー | `http://api.local:<port>/v1`（Compose の `extra_hosts` に `api.local:host-gateway` を指定） |
| LAN 上の別マシンのローカルサーバー | `http://<lan-ip>:<port>/v1` |

Docker 内では、`localhost` は Mac・Windows ホスト・Linux ホスト・LAN 上の別マシンではなく、WebUI コンテナ自身を意味します。LM Studio や Ollama がコンテナの外で動いている場合は、Docker Desktop なら `host.docker.internal` を使うか、サーバーの LAN IP アドレスを使うか、Linux Docker ホストエイリアスを追加してください:

```yaml
services:
  hermes-webui:
    extra_hosts:
      - "api.local:host-gateway"
```

その後、Base URL として `http://api.local:<port>/v1` を使います。このエイリアスは、WebUI 設定に `localhost` と書いてホストサービスではなくコンテナのループバックに解決されてしまうのを避けます。

ウィザードは保存前に `<base-url>/models` をプローブします。成功するとモデルのドロップダウンが埋まります。失敗するとセットアップステップがブロックされ、DNS 失敗、connection refused、タイムアウト、HTTP エラー、想定外のレスポンス形状などのインラインエラーが表示されます。

## ワークスペースステップ

ワークスペースは、Hermes が新しいセッションで使うべきファイルシステム上の場所です。ソースのチェックアウト、プロジェクトディレクトリ、または一般的なワークスペースフォルダで構いません。

Docker では、デフォルトの参照可能パスは `/workspace` で、これは compose ファイルがマウントするホストディレクトリにマップされます。ワークスペースが空に見える場合は、[Docker セットアップガイド](docker.md) の Docker UID/GID とマウントに関するガイダンスを確認してください。

## パスワードステップ

localhost 専用インストールではパスワード保護は任意です。WebUI を `127.0.0.1` の外、リバースプロキシの背後、または LAN に公開する場合は有効化してください。

パスワードは通常の WebUI 設定経路で保存され、サーバー側でハッシュ化されます。後から Settings で変更できます。

## 何が書き込まれるか

ウィザードは通常アプリと同じファイル・API を使います:

- 有効な Hermes `config.yaml`: プロバイダー、デフォルトモデル、該当時は Base URL。
- 有効な Hermes `.env`: 入力した場合のプロバイダー API キー。
- WebUI `settings.json`: オンボーディング完了、ワークスペース、パスワード状態、その他 WebUI 設定。

状態は通常リポジトリの外に置かれます。デフォルトでは:

- Hermes Agent の状態: Windows は `%LOCALAPPDATA%\hermes`、POSIX は `~/.hermes`
- WebUI の状態: `$HERMES_HOME/webui`（Windows デフォルト `%LOCALAPPDATA%\hermes\webui`、POSIX デフォルト `~/.hermes/webui`）

隔離したテストインストールが必要なときは、`HERMES_HOME` と `HERMES_WEBUI_STATE_DIR` でこれらを上書きしてください。

## いつ issue を立てるか

診断がローカル設定ではなく WebUI 側を指している場合に issue を立ててください。次を含めます:

1. インストール経路: ローカル bootstrap、Docker シングルコンテナ、Docker 2コンテナ、Docker 3コンテナ、WSL2、またはコミュニティ製ネイティブ Windows。
2. `/health` の出力、またはサーバーが起動しない場合は起動バナー。
3. オンボーディングで選んだプロバイダーと Base URL の形（シークレットは伏せる）。
4. Docker のプロバイダー問題では、コンテナ内部からプローブした結果。例:

```bash
docker exec hermes-webui sh -c 'curl -sS -w "\nHTTP %{http_code}\n" http://host.docker.internal:1234/v1/models | head -50'
```

5. ウィザードのインラインエラーテキストと関連ログ。

API キー、OAuth トークン、`.env` の全内容を issue に貼り付けないでください。

---

> 本ドキュメントは hermes-webui 公式リポジトリ `docs/onboarding.md` の日本語訳です。コマンド・パス・環境変数・バージョン番号は原文のまま表記しています。
