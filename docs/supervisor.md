# プロセススーパーバイザ下での Hermes Web UI 実行

Web UI をブート時に起動したい、クラッシュ時に再起動したい、他のサービスと一緒に管理したいときは、プロセススーパーバイザ（launchd、systemd、supervisord、runit、s6）を使ってください。

## TL;DR

`bootstrap.py`（または `bash start.sh`）に `--foreground` を渡します:

```bash
bash start.sh --foreground
```

または環境で `HERMES_WEBUI_FOREGROUND=1` を設定します。Web UI はフラグなしでも launchd / systemd / supervisord を自動検出しますが、明示する方が安全です。

**重要（macOS の launchd）:** `com.parantoux.hermes-webui` LaunchAgent が有効な場合、WebUI のライフサイクルは launchd を唯一の真実の源として扱ってください。同じ状態ディレクトリ/ポートに対して `./ctl.sh start`、`bash start.sh`、`python bootstrap.py`、`python server.py` を **同時に実行しないで** ください。2つ目の WebUI インスタンスが生まれ、port 8787 の再起動チャーンを引き起こしかねません。

## `--foreground` が重要な理由

これがないと、`bootstrap.py` は次を行います:

1. `server.py` を分離サブプロセスとして起動（`start_new_session=True`）
2. サーバーが起動するまで `/health` をプローブ
3. exit 0

これは対話シェル実行ではうまく動きます（`./start.sh` はサーバーをバックグラウンドで生かしたままプロンプトに戻る）。しかしプロセススーパーバイザの下では **壊れます**。スーパーバイザは追跡している PID の終了を見てジョブを完了済みとマークし、`bootstrap.py` を再起動します。再起動は port 8787 のバインドに失敗し（孤児サーバーがまだ保持している）、非ゼロで終了し、スーパーバイザが再び再起動 — ループになります。

フォアグラウンドモードでは、`bootstrap.py` はセットアップ作業を行ってから `os.execv` を呼び、自身のプロセスを `server.py` に置き換えます。スーパーバイザは長寿命のサーバーを元の子プロセスとして見ます。`KeepAlive=true` / `Restart=always` が正しく機能します。

## launchd（macOS）

`~/Library/LaunchAgents/com.example.hermes-webui.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.example.hermes-webui</string>

    <key>ProgramArguments</key>
    <array>
        <string>/bin/bash</string>
        <string>/Users/yourname/hermes-webui/start.sh</string>
        <string>--foreground</string>
    </array>

    <key>WorkingDirectory</key>
    <string>/Users/yourname/hermes-webui</string>

    <key>RunAtLoad</key>
    <true/>

    <key>KeepAlive</key>
    <true/>

    <key>StandardOutPath</key>
    <string>/Users/yourname/.hermes/webui/launchd-stdout.log</string>

    <key>StandardErrorPath</key>
    <string>/Users/yourname/.hermes/webui/launchd-stderr.log</string>

    <key>EnvironmentVariables</key>
    <dict>
        <key>HOME</key>
        <string>/Users/yourname</string>
        <key>PATH</key>
        <string>/usr/local/bin:/usr/bin:/bin</string>
    </dict>
</dict>
</plist>
```

ロード:

```bash
launchctl load ~/Library/LaunchAgents/com.example.hermes-webui.plist
launchctl print gui/$(id -u)/com.example.hermes-webui   # 状態を確認
```

plist 編集後の再ロード:

```bash
launchctl unload ~/Library/LaunchAgents/com.example.hermes-webui.plist
launchctl load   ~/Library/LaunchAgents/com.example.hermes-webui.plist
```

launchd は `XPC_SERVICE_NAME` を自動設定するため、`--foreground` 引数なしでも Web UI はフォアグラウンドモードへ自動昇格します。それでも意図のドキュメントとしてフラグの指定を推奨します。

## systemd（Linux）

`~/.config/systemd/user/hermes-webui.service`:

```ini
[Unit]
Description=Hermes Web UI
After=network.target

[Service]
Type=simple
WorkingDirectory=%h/hermes-webui
ExecStart=/bin/bash %h/hermes-webui/start.sh --foreground
Restart=on-failure
RestartSec=5

# 任意: stdout/stderr をファイルではなく journald へ
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=default.target
```

有効化＋起動:

```bash
systemctl --user daemon-reload
systemctl --user enable --now hermes-webui.service
journalctl --user -u hermes-webui.service -f
```

systemd は `INVOCATION_ID` と（stdio が journal に配線されているとき）`JOURNAL_STREAM` を設定し、どちらもフォアグラウンドモードへ自動昇格します。

## supervisord（クロスプラットフォーム）

`/etc/supervisor/conf.d/hermes-webui.conf`:

```ini
[program:hermes-webui]
command=/bin/bash /home/youruser/hermes-webui/start.sh --foreground
directory=/home/youruser/hermes-webui
user=youruser
autostart=true
autorestart=true
stopsignal=TERM
stopwaitsecs=10
stdout_logfile=/var/log/hermes-webui.out.log
stderr_logfile=/var/log/hermes-webui.err.log
environment=HOME="/home/youruser",PATH="/usr/local/bin:/usr/bin:/bin"
```

再読み込み＋起動:

```bash
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl status hermes-webui
```

supervisord は `SUPERVISOR_ENABLED` を設定し、フォアグラウンドモードへ自動昇格します。

## 自動検出される環境変数（全リスト）

これらはフラグを渡さなくても `--foreground` の挙動を引き起こします:

| 環境変数 | 設定元 | 注記 |
|---|---|---|
| `INVOCATION_ID` | systemd | サービス起動のたびに設定 |
| `JOURNAL_STREAM` | systemd | stdio が journald に配線されているとき設定 |
| `NOTIFY_SOCKET` | systemd `Type=notify` / s6 | sd_notify 形式の通知ソケット |
| `XPC_SERVICE_NAME` | launchd | plist の Label に設定 — `com.<rdns>.<svc>` 形式に絞り込み（下記参照） |
| `SUPERVISOR_ENABLED` | supervisord | supervisord 下で常に設定 |
| `HERMES_WEBUI_FOREGROUND` | あなた | 明示的なオプトイン。`1` / `true` / `yes` / `on` を受理 |

### XPC_SERVICE_NAME ノイズフィルタ

macOS の launchd は、本物のサービスだけでなく **すべての Terminal 起動シェル** で `XPC_SERVICE_NAME` を設定します。典型的なノイズ値:

- `0` — launchd の子孫一般に設定
- `application.com.apple.Terminal.<UUID>` — Terminal.app のシェル
- `application.com.googlecode.iterm2` — iTerm2
- `application.com.microsoft.VSCode` — VSCode 統合ターミナル

この変数の単なる存在チェックは、すべての Mac 開発機で対話的な `./start.sh` 実行をフォアグラウンドモードに自動昇格させ、最も一般的なインストール経路を壊してしまいます。そこで検出を launchd の **Label スタイル** 名（通常は `com.example.foo` のような reverse-DNS）に絞り込んでいます。本物の launchd plist は常にこの形式を使います。もしサービス環境で `XPC_SERVICE_NAME=0` を見かけたら、自動検出はそれを無視します。安全のため `HERMES_WEBUI_FOREGROUND=1` を設定するか `--foreground` を明示してください。

### 自動検出 *されない* スーパーバイザ

以下は、確実に検出できる環境変数を設定しません。`--foreground`（または `HERMES_WEBUI_FOREGROUND=1`）を明示してください:

- **runit**（sd_notify なし） — 純粋な runit チェーン
- **daemontools** / `svc`
- **PM2**（Python に流用されることがある Node.js プロセスマネージャ）
- **Foreman** / **Honcho**（Procfile 形式）
- 既に `exec` を使っていないカスタム CMD エントリポイントの **Docker**
- fork-and-wait する **カスタムシェルスクリプトスーパーバイザ**

スーパーバイザが自動検出リストになく、孤児 PID 再起動ループが見えたら、サービス環境に `HERMES_WEBUI_FOREGROUND=1` を設定してください。

## 診断レシピ

Web UI が再起動され続け、二重 fork ループを疑う場合:

```bash
# サーバーの稼働中 PID を確認
lsof -iTCP:8787 -sTCP:LISTEN

# その親を取得 — init（PID 1）ではなくスーパーバイザ自身であるべき
PID=$(lsof -tiTCP:8787 -sTCP:LISTEN)
ps -p "$PID" -o pid,ppid,cmd
ps -p "$(ps -o ppid= -p "$PID" | tr -d ' ')" -o pid,cmd
```

健全なフォアグラウンドモード構成は次のように見えます:

```
PID    PPID  CMD
12345  6789  /path/to/python /path/to/server.py
6789   1     /sbin/launchd        # または /usr/lib/systemd/systemd など
```

スーパーバイザであるべきところで PPID が `1`（init）の場合、孤児サーバーループが起きています。`--foreground`（または環境変数のいずれか）がプロセスに届いているか再確認してください。

## HTTP ウォッチドッグ / ディープヘルス

`KeepAlive` / `Restart=always` は、終了したプロセスしか回復できません。プロセスがまだポートで待ち受けているのにリクエスト処理が固まっている場合は、スーパーバイザを HTTP プローブと組み合わせ、プローブ失敗時に強制再起動してください。

Hermes Web UI は2レベルのヘルスを公開します:

- `/health` — `active_streams`、稼働時間、`accept_loop` ハートビートカウンタを含む安価な liveness プローブ。
- `/health?deep=1` — ストリームロックを短時間取得し、サイドバー/セッションパスを読み、projects 状態を読み、存在すれば Hermes の `state.db` に触れる readiness プローブ。ウォッチドッグにはこれを使ってください。

起動時、サーバーは `RLIMIT_NOFILE` をサポートするプラットフォームで、ファイルディスクリプタのソフト上限を 4096 に引き上げようとします。これは永続ホストの多層防御です。リークは依然修正すべきですが、ソフト上限が高ければリクエスト処理が破綻する前の診断余地が増えます。

最小限の macOS launchd ウォッチドッグスクリプト:

```bash
#!/usr/bin/env bash
set -euo pipefail
LABEL="com.example.hermes-webui"
BASE="http://127.0.0.1:8787"

if ! curl -fsS --max-time 10 "$BASE/health?deep=1" >/dev/null; then
  launchctl kickstart -k "gui/$(id -u)/$LABEL"
fi
```

別の `StartInterval` LaunchAgent から数分ごとに実行します。systemd では、同じ curl プローブを実行し失敗時に `systemctl --user restart hermes-webui.service` する timer/service ペアを推奨します。

`accept_loop.requests_total` 値は、プローブが届くと増えるはずです。プロセスが生きているのに横ばいなら、サーバーの accept ループが前進していません。バグ報告用に診断を収集しているなら、再起動前にログ/スレッドサンプルを取得してください。

---

> 本ドキュメントは hermes-webui 公式リポジトリ `docs/supervisor.md` の日本語訳です。コマンド・設定ファイル・環境変数・パスは原文のまま表記しています。
