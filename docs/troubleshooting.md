# トラブルシューティング

Hermes WebUI 運用時によくある障害モードについて、具体的な診断フローをまとめます。各項目には、症状、issue を立てる *前に* 実行すべき診断コマンド、そして過去の報告者に有効だった修正があります。

症状が一覧になく、診断でも切り分けられない場合は、https://github.com/nesquena/hermes-webui/issues にバグを登録してください。シークレット、プライベートパス、`.env` の全内容、`auth.json` の全内容、Cookie、トークン、パスワードハッシュを伏せたうえで、該当するコマンド出力を含めてください。

---

## "AIAgent not available -- check that hermes-agent is on sys.path"

**症状。** WebUI は起動しチャット画面も表示されるが、すべてのチャットリクエストがこのエラーでレスポンスまたはサーバーログに即座に失敗する。v0.51.6 以降、エラーには実行中の Python インタプリタ、該当する `sys.path` エントリ、最も一般的な修正を含む診断ブロックが含まれます。古いバージョンではメッセージは素のままです。

**原因。** WebUI はチャット時にエージェントクラスを `from run_agent import AIAgent` でインポートします。このインポートは、実行中の Python の `sys.path` に hermes-agent のチェックアウトか pip インストール済みのエージェントコピーのいずれかが含まれている場合にのみ成功します。よくある3つの失敗モード:

1. **エージェントはインストール済みだが `sys.path` 上にない。** 最も一般的。エージェントはどこか（例: `~/Programmes/hermes-agent`）にチェックアウトされているが、WebUI はそれを知らない Python で起動され、両者をリンクする `pip install -e .` がない。
2. **タイプミスや誤ったターゲットのシンボリックリンク。** エージェントへのシンボリックリンクが `ls` 上は正しく見えるが、`readlink` が存在しない、または `agent/__init__.py` を含まないパスに解決される。
3. **`HERMES_WEBUI_AGENT_DIR` が誤ったディレクトリに設定されている。** 上書き環境変数は自動検出に優先し、エージェントコードのないディレクトリを指している。

### Step 1 — エージェントの場所を確認する

```bash
# ~/hermes-agent（デフォルトの場所）がある場合:
ls -la ~/hermes-agent
readlink ~/hermes-agent          # シンボリックリンクなら、どこに解決されるか？
ls ~/hermes-agent/agent/__init__.py 2>&1
```

3番目のコマンドは成功しなければなりません（ファイルが存在すること）。失敗する場合、シンボリックリンクが壊れているか、エージェントモジュールを欠くディレクトリを指しています。まずそれを直してください。

### Step 2 — WebUI が正しい Python を使っているか確認する

```bash
cd ~/hermes-webui && ./start.sh 2>&1 | grep -iE 'agent|python|hermes_webui_python' | head -20
```

起動バナーは、どの Python とエージェントディレクトリを解決したかを表示します。エージェントディレクトリが空か Python が誤っている場合は、上書きを設定します:

```bash
export HERMES_WEBUI_AGENT_DIR=/absolute/path/to/hermes-agent
export HERMES_WEBUI_PYTHON=/absolute/path/to/agent/venv/bin/python
./start.sh
```

### Step 3 — エージェントを editable モードでインストールする

これは最も一般的な修正で、元の issue #1695 を解決します:

```bash
cd /path/to/hermes-agent          # pyproject.toml と agent/ モジュールを含むディレクトリ
pip install -e .                  # WebUI を動かすのと同じ python を使う
```

その後、WebUI を再起動します:

```bash
cd ~/hermes-webui
./start.sh
```

### Step 4 — 手動インポートで検証する

Step 1〜3 でも動かない場合、WebUI の Python がそもそもエージェントをインポートできるか確認します:

```bash
$HERMES_WEBUI_PYTHON -c "from run_agent import AIAgent; print('ok')" 2>&1
```

（環境変数が未設定なら `$HERMES_WEBUI_PYTHON` を Step 2 の実際の Python パスに置き換えてください。）これが `ok` を表示すれば、その Python の `sys.path` にエージェントは載っており、WebUI は動くはずです。

これが失敗する場合、`import run_agent` 自体が壊れています。エージェントの pyproject.toml が `run_agent` をトップレベルモジュールとして列挙しているか、エージェントディレクトリが PYTHONPATH 上にあるか確認してください:

```bash
PYTHONPATH=/path/to/hermes-agent $HERMES_WEBUI_PYTHON -c "from run_agent import AIAgent; print('ok')"
```

PYTHONPATH を追加すると直る場合は、`pip install -e .`（推奨）か、`HERMES_WEBUI_AGENT_DIR` をそのディレクトリに設定して、パスを永続化してください。

### いつバグを立てるか

Step 1〜4 を実行してもインポートが依然失敗し、*かつ* `pip install -e .` が成功し、*かつ* `PYTHONPATH=... python -c "from run_agent import AIAgent"` が成功する場合は、本物の WebUI バグです。https://github.com/nesquena/hermes-webui/issues に次を添えて登録してください:

- Step 1〜4 の各コマンドの出力
- WebUI の `ImportError` が表示する完全な診断ブロック（v0.51.6+）
- OS、Python バージョン、エージェントのインストール方法

---

## "Response interrupted." マーカーが "no agent output was recovered" と言い続ける

**症状。** ライブレスポンスのストリームがターン完了前に停止した後（手動再起動、OOM、クラッシュ、ブラウザ/SSE 切断、ワーカー記録の喪失など）、該当チャットに `**Response interrupted.**` マーカーが表示される。そのターンの run-journal がディスク上で既に見えていれば、マーカーは部分出力が復元されたと表示します。そうでなければ、ユーザーのターンを保持し、エージェント出力はまだ復元されていないと表示します。

**理由。** サイドカー修復は、古いストリームを検出した後に run-journal を再チェックし、その結果をワンショットのシグナルとして使います。WSL2（9p / DrvFs）や一部のネットワークバックエンド構成では、run-journal の `.jsonl` は停止したワーカーが書き込むものの、WebUI プロセスはそれらの書き込みをまだ見ていないページキャッシュ状態を通して読むため、復元が「空」を返し、本来ならマーカーが恒久的に焼き付いてしまいます。修正は *遅延（lazy）* リトライ経路を導入します。サイドカー修復が可視の出力を読めないがストリーム ID は分かっている場合、マーカーに `_pending_journal_recovery` フラグを保存し、journal が読めるようになるまで（またはリトライ予算が尽きるまで）`get_session()` から復元を再試行します。

**割り込みの分類。** WebUI は、あらゆる古いストリームを再起動だと示唆するのではなく、ユーザー向けのケースを区別して保持するようになりました:

- **Browser/SSE connection interrupted** — ライブブラウザの `EventSource` トランスポートが切断された。UI は `Connection interrupted` を報告し、最終的なブラウザ側通知を表示する前に status/replay/session restore を試みます。チャットおよび gateway の SSE エラーは、サニタイズ済みの小さな診断イベントを `/api/client-events/log` に POST します（source、session id、stream id、readyState、visibility、online state、クエリ文字列を除いた path）。これによりサーバーログがブラウザのトランスポート喪失とバックエンドワーカー喪失を区別できます。
- **Lost worker bookkeeping** — stream id が失われ、ワーカーレジストリにアクティブな run がもうない。復元マーカーは `interruption_cause: "lost_worker_bookkeeping"` を持ち、`/api/chat/stream/status` は、もうアクティブでない非終端 journal に対して `terminal_state: "lost-worker-bookkeeping"` を報告します。
- **Stream/run split-brain** — ストリームは失われたが `ACTIVE_RUNS` がまだワーカーを列挙している。復元マーカーは `interruption_cause: "stream_run_split_brain"` を持ち、トランスクリプトは再起動ではなく記録のスプリットブレインだと示します。
- **Process crash/restart** — `SERVER_START_TIME` が `pending_started_at` より新しく、WebUI プロセスがターン開始後に起動したことを意味する。復元マーカーは `interruption_cause: "process_restart"` を持ち、プロセス起動の証拠がクラッシュまたは再起動を指すと明示します。

**診断。**

以下のディスク上の場所は、デフォルトの `~/.hermes/webui` 状態ディレクトリを前提とします。`HERMES_WEBUI_STATE_DIR` で上書きしている場合は、各ステップの `~/.hermes/webui` をそのパスに置き換えてください。

1. マーカーから該当する session id と stream id を特定します。マーカー JSON は `~/.hermes/webui/sessions/<sid>.json` にあります。修正後は `_journal_retry_stream_id` キーに表示されます。修正前のセッションはレガシーな文言のみで、リトライメタはありません。
2. run-journal が実イベントを含むか確認します:
   ```bash
   ls -la ~/.hermes/webui/sessions/_run_journal/<sid>/<stream_id>.jsonl
   head -2 ~/.hermes/webui/sessions/_run_journal/<sid>/<stream_id>.jsonl
   ```
   ファイルが存在し `token` / `tool` イベントを含む場合、次にそのセッションを開いたときに遅延リトライ経路がそれらを拾います。

**修正。** ブラウザでセッションを再読み込みします。次の `get_session()` 呼び出しでマーカーが再評価され、journal 化されたイベントがディスク上で可視なら、マーカーは *「上の部分出力は run journal から復元されました…」* の文言に昇格し、journal 化されたアシスタントテキスト＋ツールカードが時系列順にマーカーの上に配置されます。手動でのサイドカー編集は不要です。

**トリガー。** サイドバーのメタデータポーリングは、この自己修復を実行するには意図的に不十分です。`/api/session?messages=0&resolve_model=0` のようなリクエストは `metadata_only=True` でセッションを読み込み、完全なメッセージ配列をスキップするため、遅延 journal リトライヘルパもスキップします。該当する会話をクリック/開いて、メッセージパネルに完全な `messages=1` ロードを実行させてください。その完全描画こそが journal を再チェックし、マーカーを昇格できます。

**上限。** 遅延リトライ経路は、12回の失敗または実時間 24h の経過で諦め、その時点でマーカーは中立的な *「部分出力は失われた可能性があります。」* の文言に降格されます。これにより、本当に失われた journal に対して「再読み込みで再試行」のプロンプトが永遠に残らないようにします。

**いつバグを立てるか。** 修正後に遅延リトライの文言（*「run journal から部分出力を復元中 — このセッションを再読み込みして再試行してください。」*）が表示されるのに、`.jsonl` が明らかに `token` イベントを含むのにセッションを再読み込みしても復元済み文言へ昇格しない場合は、マーカー JSON と run-journal ファイルを取得してバグを登録してください。

---

## その他のトラブルシューティング

このドキュメントは時間とともに増えていきます。繰り返し起こる障害モードがまだ載っていなければ、PR で追加してください。各項目のフォーマットは: **症状 → 理由 → 診断コマンド → 修正 → いつバグを立てるか**。

関連リファレンス:

- [`docs/supervisor.md`](supervisor.md) — プロセススーパーバイザ設定（launchd、systemd、supervisord、runit/s6）。bootstrap の supervisor-foreground フラグを含む。
- [`docs/docker.md`](docker.md) — Docker compose セットアップ、よくある障害モード、bind-mount 移行。
- [`docs/wsl-autostart.md`](wsl-autostart.md) — Windows でのログイン時 WSL2 自動起動。
- [`docs/EXTENSIONS.md`](EXTENSIONS.md) — WebUI 拡張の注入、セキュリティモデル、例。

---

> 本ドキュメントは hermes-webui 公式リポジトリ `docs/troubleshooting.md` の日本語訳です。コマンド・パス・APIエンドポイント・識別子・バージョン番号は原文のまま表記しています。
