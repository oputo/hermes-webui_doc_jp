# エージェント支援オンボーディングチェックリスト

このチェックリストは、人間が Hermes WebUI のオンボーディングをインストール・再インストール・デバッグするのを手伝う AI アシスタント向けです。人間の初回起動ウィザードを置き換えるものではありません。bootstrap コマンドの実行、ログ調査、クリーンアップ経路の推奨の前に使ってください。

AI アシスタントなら、オンボーディング、bootstrap、プロバイダー設定、再インストール、初回起動サポートを手伝う前にこのファイルを読んでください。

## 役割分担

人間オペレーターが所有するもの:

- インストール経路の選択
- プロバイダーとモデルの選択
- API キー、OAuth コード、パスワードの入力
- 実 Hermes home のクリーンアップの承認
- localhost 外への外部公開の承認

アシスタントが所有するもの:

- 人間が明示的に別途言わない限り、隔離した試行ディレクトリを使う
- 非シークレットのステータスエンドポイントとログを確認
- どのステップが通過/失敗したかを説明
- Discord や GitHub サポート用に伏字化したエビデンスを収集
- 破壊的クリーンアップ、認証情報の取り扱い、公開露出の前に停止

## 厳格な安全ルール

- 人間がその正確なアクションを明示的に求めない限り、実 `~/.hermes` ディレクトリを削除・移動・上書きしない。
- API キー、OAuth トークン、Cookie、`.env` の全内容、`auth.json` の全内容、パスワードハッシュを出力しない。
- オンボーディング試行中に実 cron ジョブ、実セッション、実プロファイル、実メモリファイルを変更しない。
- パスワード保護と明示的な人間の承認なしに WebUI を公開インターフェースに露出しない。
- `localhost`、`127.0.0.1`、プライベート LAN アドレス、Docker コンテナのループバックパスのようなローカルサービスチェックをプロキシ・トンネルしない。

## プリフライト

基本コンテキストを確認:

```bash
pwd
git branch --show-current
git rev-parse --short HEAD
python3 --version
```

リポジトリローカルの環境上書きが bootstrap に影響するか確認:

```bash
test -f .env && grep -n 'HERMES_HOME\|HERMES_WEBUI_STATE_DIR\|HERMES_WEBUI_PORT\|HERMES_WEBUI_HOST' .env
```

`.env` が存在する場合、ファイル全体を出力しない。アクティブな Hermes home、WebUI 状態ディレクトリ、ポート、ホストを理解するのに必要な特定の非シークレットキーのみを調べる。

## 隔離されたローカル試行

再インストールやサポート試行には、隔離した Hermes home と WebUI 状態ディレクトリを使う。これでテストをオペレーターの実メモリ、セッション、プロファイル、認証情報、cron 状態から遠ざけます。

```bash
mkdir -p ~/hermes-onboarding-test
HERMES_HOME=~/hermes-onboarding-test/.hermes \
HERMES_WEBUI_STATE_DIR=~/hermes-onboarding-test/webui \
HERMES_WEBUI_PORT=8789 \
python3 bootstrap.py
```

開く:

```text
http://127.0.0.1:8789
```

bootstrap は選択された WebUI 状態ディレクトリ下にポート固有のログを書きます:

```text
~/hermes-onboarding-test/webui/bootstrap-8789.log
```

デーモン形式のインストールでは、`ctl.sh` がデフォルトでアクティブな `HERMES_HOME` にデーモンログを書きます:

```text
~/.hermes/webui.log
```

隔離試行環境を使うときは、人間が特に `ctl.sh` を検証したい場合を除き、上記の bootstrap コマンドを優先してください。

## 非シークレットのエビデンスコマンド

サーバー起動後、シークレットなしでステータスを収集:

```bash
curl -sS http://127.0.0.1:8789/health
curl -sS http://127.0.0.1:8789/api/onboarding/status
find ~/hermes-onboarding-test -maxdepth 3 -type f | sort
tail -n 120 ~/hermes-onboarding-test/webui/bootstrap-8789.log
```

`/api/onboarding/status` を要約するとき、次に焦点を当てる:

- `completed`
- `system.hermes_found`
- `system.imports_ok`
- `system.config_path`
- `system.config_exists`
- `system.setup_state`
- `system.provider_configured`
- `system.provider_ready`
- `system.chat_ready`
- `system.current_provider`
- `system.current_model`
- `system.current_base_url`
- `system.env_path`

予期しない機密のローカルパスや値を含む場合、完全なペイロードを貼り付けない。人間が公開 GitHub や Discord のサポートレポートを求めるとき、パスとプロバイダー詳細を伏字化する。

## 合格基準

ローカルオンボーディング試行が合格するのは:

- `/health` が正常に返る。
- `/api/onboarding/status` が JSON を返す。
- `completed` が false のときウィザードが現れる。
- `completed` が true、または `HERMES_WEBUI_SKIP_ONBOARDING=1` が意図的に設定されているとき、ウィザードが邪魔にならない。
- `system.hermes_found` と `system.imports_ok` が期待される bootstrap 状態に一致。
- 人間がチャットをサポートすべきプロバイダー経路を完了した後、`system.provider_ready` と `system.chat_ready` が true になる。
- 試行中、`system.config_path` と `system.env_path` が意図した隔離 `HERMES_HOME` 内を指す。
- WebUI ファイルが意図した `HERMES_WEBUI_STATE_DIR` 下に書かれる。

人間が CLI で完了する必要のあるプロバイダーを選んだ場合、合格とは、ウィザードがブラウザで非対応の認証情報を集めようとするのではなく、`hermes model` や `hermes auth` へ正しく案内することを意味し得ます。

## 失敗トリアージ

サーバーが起動しない場合:

- bootstrap ログを確認
- `8789` のポート競合を確認
- Python が `bootstrap.py` を実行できるか確認
- `.env` が隔離ディレクトリやポートを上書きしていないか確認

オンボーディングが `agent_unavailable` を報告する場合:

- bootstrap が Hermes Agent を見つけたかインストールしたか確認
- 実行中の Python が `run_agent.AIAgent` をインポートできるか確認
- `docs/troubleshooting.md`、特に `AIAgent not available` フローを使う

オンボーディングが `provider_incomplete` を報告する場合:

- プロバイダーが API キー型か OAuth 型かローカルか確認
- 人間に認証情報を入力させるか CLI 認証フローを実行させる
- 人間にシークレットをチャットへ貼り付けるよう求めない

ローカルモデルサーバーが正常にプローブできない場合:

- ネイティブ macOS/Linux から、サーバーが同一ホストなら `http://127.0.0.1:<port>/v1` を使う
- Docker Desktop から、`http://host.docker.internal:<port>/v1` を使う
- LAN 上の別マシンから、サーバーの LAN IP と `/v1` を使う
- コンテナ内の `localhost` はコンテナ自身であることを覚えておく

パスワードやリバースプロキシ挙動が紛らわしい場合:

- 最初のパスは `127.0.0.1` に保つ
- WebUI を localhost の外に露出する前にパスワード保護を要求
- トークンや Cookie を貼り付けずにリバースプロキシの形をサポートレポートに含める

## 最終サポートレポート

人間、Discord、GitHub に結果を報告するとき、この形を使う:

```text
Install path:
OS / Python:
Repo commit:
Command used:
WebUI URL:
State isolation:
Health result:
Onboarding status summary:
Files created or changed:
Log excerpt:
Pass/fail:
Next recommended action:
```

公開投稿の前にシークレットとプライベートパスを伏字化してください。

---

> 本ドキュメントは hermes-webui 公式リポジトリ `docs/onboarding-agent-checklist.md` の日本語訳です。コマンド・APIフィールド名・環境変数・パス・レポートテンプレート（コードブロック内）は原文のまま表記しています。
