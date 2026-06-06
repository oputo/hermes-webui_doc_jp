# Windows / WSL 自動起動

Hermes WebUI は WSL2 上でよく動きますが、ネイティブ Windows のログインでは Linux のユーザープロセスは自動的に起動しません。このガイドは、サポートされた2つの選択肢を扱います:

1. **WSL セッション起動** — シンプルで低リスク。次に WSL シェルを開いたときに WebUI が起動します。
2. **Windows タスクスケジューラ** — 真の Windows ログオン起動。Windows が `wsl.exe` を呼び出し、それが WSL 起動スクリプトを実行します。

どちらの経路も同じ WSL 起動スクリプトを使います:

```text
scripts/wsl/hermes_webui_autostart.sh
```

このスクリプトは繰り返し呼び出しても安全です。ロックファイルを使い、`/health` エンドポイントを確認し、pid ファイルを確認し、ログを書いてから `start.sh --foreground` をバックグラウンドで起動します。ユーザーパスをハードコードせず、デフォルトでは自身の場所からリポジトリルートを導出します。

## スクリプト設定

WSL ランチャーは次の環境変数に対応します:

| 変数 | デフォルト | 目的 |
|---|---|---|
| `HERMES_WEBUI_REPO` | スクリプトを含むリポジトリ | 起動する WebUI チェックアウト |
| `HERMES_WEBUI_LOG_DIR` | `$HOME/.hermes/webui/logs` | 自動起動と WebUI のログ |
| `HERMES_WEBUI_HOST` | `127.0.0.1` | `start.sh` / `bootstrap.py` に渡すホスト |
| `HERMES_WEBUI_PORT` | `8787` | WebUI ポートとヘルスチェックポート |
| `HERMES_WEBUI_HEALTH_URL` | `http://127.0.0.1:$HERMES_WEBUI_PORT/health` | WebUI が既に動いているか判定する URL |
| `HERMES_WEBUI_PID_FILE` | `$HERMES_WEBUI_LOG_DIR/hermes-webui.pid` | 重複防止に使う pid ファイル |
| `HERMES_WEBUI_REQUIRE_AGENT_PROCESS` | `0` | 任意: WebUI 起動前に別の Hermes プロセスが必要なローカル構成のときのみ `1` にする |

WSL 内で一度スクリプトに実行権を付与します:

```bash
cd /path/to/hermes-webui
chmod +x scripts/wsl/hermes_webui_autostart.sh
```

パスとログを検証するため手動実行します:

```bash
scripts/wsl/hermes_webui_autostart.sh
curl -fsS http://127.0.0.1:8787/health
```

ログの書き込み先:

```text
$HOME/.hermes/webui/logs/webui_autostart.log
$HOME/.hermes/webui/logs/hermes_webui.log
```

## 選択肢1: WSL セッション起動

これは WSL ログインシェルの起動時に WebUI を起動します。日中に既に WSL を開く習慣があるなら、最も手軽な選択肢です。

WSL 内の `~/.profile` か `~/.bashrc` に、リポジトリパスを調整して次を追加します:

```bash
if [ -x "$HOME/hermes-webui/scripts/wsl/hermes_webui_autostart.sh" ]; then
  HERMES_WEBUI_REPO="$HOME/hermes-webui" \
    "$HOME/hermes-webui/scripts/wsl/hermes_webui_autostart.sh" >/dev/null 2>&1 &
fi
```

新しい WSL ターミナルを開いて確認します:

```bash
curl -fsS http://127.0.0.1:8787/health
```

複数の WSL ターミナルを開いても、ロック・ヘルスチェック・pid ファイルがすべて「既に起動済み」に収束するため、ランチャーは WebUI プロセスを1つだけ起動するはずです。

## 選択肢2: Windows タスクスケジューラ起動

WSL ターミナルを開く前でも、Windows ログオン時に WebUI を自動起動させたい場合に使います。

ヘルパー PowerShell スクリプト:

```text
scripts/windows/setup_webui_autostart.ps1
```

Windows PowerShell から、起動スクリプトの WSL パスを指定して実行します:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
.\scripts\windows\setup_webui_autostart.ps1 `
  -WslScriptPath "/home/your-user/hermes-webui/scripts/wsl/hermes_webui_autostart.sh" `
  -Distro "Ubuntu"
```

注記:

- `-Distro` は任意。省略するとデフォルトの WSL ディストロを使います。
- デフォルトのタスク名は `HermesWebUIAutoStart`。別名が必要なら `-TaskName` を渡します。
- スクリプトは冪等です。再実行は重複を作らず既存のスケジュールタスクを更新します。
- タスクはログオン時に現在の Windows ユーザーとして最小権限で動きます。
- スケジュールタスク登録をプレビューするには `-WhatIf` を追加します。
- 登録直後にタスクを開始するには `-RunNow` を追加します。
- WSL パスが存在する前にタスクを登録する必要があるときのみ `-SkipValidation` を追加します。

後でタスクを確認・削除するには:

```powershell
Get-ScheduledTask -TaskName HermesWebUIAutoStart
Unregister-ScheduledTask -TaskName HermesWebUIAutoStart -Confirm:$false
```

## トラブルシューティング

まず WSL のログを確認します:

```bash
tail -n 80 "$HOME/.hermes/webui/logs/webui_autostart.log"
tail -n 80 "$HOME/.hermes/webui/logs/hermes_webui.log"
```

よくある原因:

| 症状 | 考えられる原因 | 修正 |
|---|---|---|
| タスクは存在するが WebUI に到達できない | 選択したディストロに対し WSL スクリプトパスが誤り | 正しい `-WslScriptPath` と `-Distro` で PowerShell セットアップを再実行 |
| WSL を開いた後にしか WebUI が起動しない | タスクスケジューラではなく WSL セッション起動を使った | Windows のスケジュールタスクをインストール |
| 複数のログインイベントが短時間に発生 | 通常の Windows 起動挙動 | WSL スクリプトは `already running` をログし、重複プロセスを避けるはず |
| ヘルスチェックは失敗するが pid は存在 | WebUI がまだ起動中、またはポートが異なる | `HERMES_WEBUI_PORT` と `hermes_webui.log` を確認 |

代わりに WSL2 の systemd 統合が欲しい場合は、フォアグラウンドのプロセススーパーバイザガイダンスについて `docs/supervisor.md` を参照し、Linux の `systemd --user` パターンを自分のディストロに適応させてください。

---

> 本ドキュメントは hermes-webui 公式リポジトリ `docs/wsl-autostart.md` の日本語訳です。コマンド・パス・環境変数は原文のまま表記しています。
