# Hermes Web UI: 開発者・アーキテクチャガイド

> 本ドキュメントは、Hermes Web UI に取り組むすべての人（人間でもエージェントでも）にとっての正典リファレンスです。コードの正確な現状、開発中に発見されたあらゆる設計判断とクセ、そして ROADMAP.md の機能ロードマップと並行して進むフェーズ分けされたアーキテクチャ改善ロードマップを扱います。
>
> アーキテクチャ変更が行われたら、本ドキュメントを更新し続けてください。

> 現在の出荷ビルド: `v0.51.192`（2026年5月31日）。
> 自動カバレッジ: `pytest tests/ --collect-only -q` で約 7,150 テスト。CI は Python 3.11、3.12、3.13（各3並列シャード）で全 PR に対して実行され、加えて ruff lint ゲート、ヘッドレスブラウザスモークテスト、Docker スモークテストがあります。
>
> 注目すべきアーキテクチャ状態: bootstrap と初回オンボーディングフローがセットアップ検出を担う。デフォルトの WebUI 状態ディレクトリは `~/.hermes/webui`。`ctl.sh` がホームラボインストール向けのデーモンラッパーを提供。チャットストリーミングは依然 WebUI 所有の SSE で、ストリーム所有権ガード、キャンセル、非同期手動圧縮、turn-journal 監査の配管を備える。プロバイダー/モデル検出はプロファイル対応で、ライブモデルキャッシュ無効化とカスタムプロバイダースコープを持つ。（上記のバージョン/テスト数は定期的なスナップショット。権威ある情報源は最新の git タグと `pytest --collect-only`。）

---

## 1. 概要と目的

Hermes Web UI は、CLI と機能的に同等な、Hermes エージェントへのブラウザベースのインターフェースを与える軽量 Web アプリケーションです。Claude 風のインターフェースをモデルにしています。セッション管理用のサイドバー、中央のチャットエリア、ワークスペース参照とプレビューサーフェスに使うデマンド駆動の右パネル。右パネルはデスクトップではデフォルトで閉じており、コンテンツの参照やプレビューに実際に使うときのみ開きます。

再読み込み時に最初の描画の不一致が見えるのを防ぐため、`static/index.html` は、メインスタイルシートが読み込まれる前に、保存済みのワークスペースパネル状態を `document.documentElement.dataset.workspacePanel` にプリロードします。デスクトップ CSS はそのプリロードマーカーを即座に尊重し、`static/boot.js` がそのデータセットをランタイムのパネル状態機械と同期し続けます。

設計哲学は意図的にミニマルです。ビルドステップなし、バンドラなし、フロントエンドフレームワークなし。Python サーバーはルーティングシェル（server.py）とビジネスロジックモジュール（api/）に分割されています。フロントエンドは static/ から読み込まれる7つの vanilla JS モジュールです。これにより、ターミナルから、またはエージェントによってコードを簡単に変更できます。

Hermes レベルのクロムは意図的に集約されています。サイドバーには専用のブランドヘッダがありません。代わりにフッターが、グローバル設定・会話のインポート/エクスポート・会話クリアアクション用の1つのタブ付きコントロールセンターモーダルを開く、単一の「Hermes WebUI」起動ボタンを公開します。トップバーは会話コンテキストとワークスペース/ファイルのトグルに集中したままです。

---

## 2. ファイルインベントリ

    <repo>/
    server.py              薄いルーティングシェル ＋ HTTP Handler ＋ 認証ミドルウェア。
                           全ルート処理を api/routes.py に委譲。
    bootstrap.py           ワンショットランチャー: 任意のエージェントインストール、依存、ヘルス待ち、ブラウザ起動。
    start.sh               シェルベース起動用の bootstrap.py の薄いラッパー。
    ctl.sh                 ホームラボインストール向けデーモンライフサイクルラッパー（start/stop/restart/status/logs）。
    pyproject.toml         ツール設定（ruff lint ゲート）。パッケージ配布物ではない。
    Dockerfile             python:3.12-slim コンテナイメージ
    docker-compose.yml     名前付きボリュームと任意認証を持つ Compose 設定
    .dockerignore          Docker ビルドから .git、tests/、.env* を除外
    api/
      __init__.py          パッケージマーカー
      auth.py              任意のパスワード認証、署名 Cookie、passkeys/WebAuthn
      config.py            検出、グローバル、モデル検出、再読み込み可能な config
      helpers.py           HTTP ヘルパー: j()、bad()、require()、safe_resolve()、セキュリティヘッダ
      models.py            Session モデル ＋ CRUD、セッションごとのプロファイル追跡、CLI/state.db ブリッジ
      profiles.py          プロファイル状態管理、hermes_cli ラッパー
      onboarding.py        初回オンボーディング状態、実プロバイダー設定の書き込み、OAuth リンク、準備完了検出
      routes.py            全 GET ＋ POST ルートハンドラ（if/elif ディスパッチ、デコレータなし）
      startup.py           起動ヘルパー: auto_install_agent_deps()
      state_sync.py        /insights 同期 — message_count をエージェントの state.db へ
      streaming.py         SSE エンジン、run_agent、cancel、compression、HERMES_HOME の save/restore
      updates.py           自己更新チェックとリリースノート
      upload.py            Multipart パーサ、ファイルアップロードハンドラ
      workspace.py         ファイル操作: list_dir、read_file_content、git 検出、ワークスペースヘルパー
    static/
      index.html           HTML テンプレート
      style.css            モバイルレスポンシブ、テーマ ＋ スキン、KaTeX を含む全 CSS
      ui.js                DOM ヘルパー、renderMd、ツールカード、コンテキストインジケータ、ファイルツリー
      workspace.js         ファイルプレビュー、ファイル操作、git バッジ、中央 api() fetch ラッパー
      sessions.js          セッション CRUD、一覧描画、折りたたみグループ、検索、SSE 同期
      messages.js          send()、SSE イベントハンドラ、approval/clarify、トランスクリプト、復旧
      panels.js            Cron、skills、memory、profiles、todo、settings（Control Center）
      commands.js          スラッシュコマンドレジストリ、パーサ、オートコンプリートドロップダウン
      boot.js              イベント配線、モバイルナビ、音声入力、テーマ/スキンブート、bfcache ハンドラ
      onboarding.js        初回ウィザードオーバーレイ、プロバイダー設定フロー
      i18n.js              ローカライズカタログ（en、es、de、zh、zh-Hant、ru、…）
      login.js             ログインページ ＋ open-redirect ガード
      icons.js             Lucide アイコンパスレジストリ
      sw.js                サービスワーカー: オフラインシェルキャッシュ、バージョン固定アセット
    tests/
      conftest.py          隔離されたテストサーバー/状態フィクスチャ
      ~700 test files      pytest で収集される約 7,150 テスト（正確には `pytest --collect-only -q`）
      test_regressions.py  恒久的な回帰ゲート
    CONTRIBUTING.md        コントリビューターワークフローと PR への期待。
    ROADMAP.md             機能・プロダクトロードマップ文書。
    SPRINTS.md             CLI ＋ Claude 同等性目標を伴う今後のスプリント計画。
    ARCHITECTURE.md        このファイル。
    TESTING.md             手動ブラウザテスト計画と自動カバレッジリファレンス。
    CHANGELOG.md           バージョンごとのリリースノート。
    CONTRIBUTORS.md        コミュニティクレジット（メンテナワークスペーススクリプトで再生成）。
    requirements.txt       Python 依存。
    .env.example           環境変数上書きのサンプル。

> ファイルごとの行数は意図的に省略 — リリースごとにずれます。現在のサイズは `git ls-files | xargs wc -l`（またはエディタ）を使ってください。上記の各ファイルの役割が永続的な部分です。

状態ディレクトリ（ランタイムデータ、ソースとは別）:

    ~/.hermes/webui/
    sessions/          セッションごとに1つの JSON ファイル: {session_id}.json
    workspaces.json    登録済みワークスペース一覧
    last_workspace.txt 最後に使ったワークスペースパス
    settings.json      ユーザー設定（デフォルトモデル、ワークスペース、送信キー、パスワードハッシュ）
    projects.json      セッションプロジェクトグループ（name、color、id）

ログファイル:

    ~/.hermes/webui/bootstrap-8787.log   start.sh/bootstrap のバックグラウンドサーバーログ
    ~/.hermes/webui.log                  ctl.sh デーモンログ

---

## 3. ランタイム環境

- Python インタプリタ: <agent-dir>/venv/bin/python
- venv はすべての Hermes エージェント依存を持つ（run_agent、tools/*、cron/*）
- サーバーは 127.0.0.1:8787 にバインド（localhost のみ、公開インターネットではない）
- Mac からのアクセス: SSH トンネル: ssh -N -L 8787:127.0.0.1:8787 <user>@<your-server>
- サーバーは sys.path.insert(0, parent_dir) で Hermes モジュールをインポート

挙動を制御する環境変数:

    HERMES_WEBUI_HOST              バインドアドレス（デフォルト: 127.0.0.1）
    HERMES_WEBUI_PORT              ポート（デフォルト: 8787）
    HERMES_WEBUI_DEFAULT_WORKSPACE 新規セッションのデフォルトワークスペースパス
    HERMES_WEBUI_STATE_DIR         sessions/ フォルダの場所
    HERMES_CONFIG_PATH             ~/.hermes/config.yaml へのパス
    HERMES_WEBUI_DEFAULT_MODEL     任意のモデル上書き。未設定はプロバイダーデフォルト
    HERMES_WEBUI_PASSWORD          任意: パスワード認証を有効化（デフォルトオフ）
    HERMES_WEBUI_SKIP_ONBOARDING   任意: 初回オンボーディングウィザードをバイパス
    HERMES_PREFILL_MESSAGES_FILE   ブラウザターン prefill コンテキスト用の任意 JSON メッセージリスト
    HERMES_WEBUI_PREFILL_MESSAGES_SCRIPT JSON メッセージまたはプレーンテキストのユーザー prefill コンテキストを出力する任意コマンド
    HERMES_WEBUI_PREFILL_MESSAGES_SCRIPT_TIMEOUT 任意のスクリプトタイムアウト（秒、デフォルト 5、最大 30）
    HERMES_WEBUI_PREFILL_CONTEXT_MAX_CHARS パース済み prefill 予算（文字、デフォルト 12000、0 で無効）
    HERMES_HOME                    Hermes 状態のベースディレクトリ（デフォルト ~/.hermes）

テスト隔離用の環境変数（conftest.py が設定）:

    HERMES_WEBUI_TEST_PORT=...                         任意の固定テストポート
    HERMES_WEBUI_TEST_STATE_DIR=~/.hermes/webui-test-* 任意の固定テスト状態
    HERMES_WEBUI_DEFAULT_WORKSPACE=.../test-workspace  隔離されたテストワークスペース

テストは本番サーバー（port 8787）と決して通信しません。
テスト状態ディレクトリは各テストセッション前にワイプされ、後に削除されます。
参照: <repo>/tests/conftest.py

リクエストごとの環境変数（チャットハンドラが設定し、後で復元）:

    TERMINAL_CWD         エージェント実行前に session.workspace に設定。
                         terminal ツールがこれを読みデフォルト cwd にする。
    HERMES_EXEC_ASK      危険なコマンドの承認ゲートを有効化するため "1" に設定。
    HERMES_SESSION_KEY   session_id に設定。approval ツールはこの値で保留エントリを
                         キー付けし、セッションごとの承認状態を可能にする。
    HERMES_HOME          エージェント実行前にアクティブプロファイルのディレクトリに設定。
                         各エージェント実行の前後で保存・復元される。

警告: これらの環境変数はプロセスグローバルです。2つの並行チャットリクエストは互いを上書きします。これは単一ユーザー・単一並行リクエストの利用でのみ安全です。修正については Architecture Phase B を参照。

---

## 4. サーバーアーキテクチャ: 現状

### 4.1 HTTP サーバー層

Python 標準ライブラリの ThreadingHTTPServer（http.server より）。各 HTTP リクエストは独自のスレッドで動きます。Handler クラスは BaseHTTPRequestHandler をサブクラス化し、2つのメソッドを持ちます:

    do_GET    ルート: /、/health、/api/session、/api/sessions、/api/list、
                      /api/chat/stream、/api/file、/api/approval/pending、
                      /api/session/worktree/status
    do_POST   ルート: /api/upload、/api/session/new、/api/session/update、
                      /api/session/delete、/api/chat/start、/api/chat、
                      /api/approval/respond、/api/session/worktree/remove

ルーティングは各メソッド内のフラットな if/elif チェーンです。ルーティングフレームワークはありません。

全ハンドラが使うヘルパー関数:

    j(handler, payload, status=200)     正しいヘッダで JSON レスポンスを送る
    t(handler, payload, status=200, ct) プレーンテキストまたは HTML レスポンスを送る
    read_body(handler)                  POST ボディを読み JSON パースする

do_POST における重要な順序ルール:
/api/upload のチェックは read_body() を呼ぶ **前** になければなりません。read_body() は handler.rfile.read() を呼び、HTTP ボディストリームを消費します。アップロードハンドラも rfile（multipart ペイロードの読み取り）を必要とします。multipart リクエストで read_body() が先に動くと、アップロードハンドラは空のボディを受け取り、アップロードが黙って失敗します。

### 4.2 Session モデル

Session は素の Python クラスです（dataclass でも SQLAlchemy でもない）:

    フィールド:
      session_id    16進文字列、12文字（uuid4().hex[:12]）
      title         文字列、最初のユーザーメッセージから自動設定
      workspace     絶対パス文字列、作成時に解決
      model         モデル ID 文字列（例: "anthropic/claude-sonnet-4.6"）
      messages      OpenAI 形式のメッセージ dict のリスト
      created_at    float Unix タイムスタンプ
      updated_at    float Unix タイムスタンプ、save() のたびに更新
      pinned        bool、デフォルト False（Sprint 12）
      archived      bool、デフォルト False（Sprint 14）
      project_id    文字列または null、projects.json への FK（Sprint 15）
      tool_calls    ツール呼び出し dict のリスト（Sprint 10）

    主要メソッド:
      path（プロパティ）  SESSION_DIR/{session_id}.json を返す
      save()           __dict__ を整形 JSON として path に書き、updated_at を更新
      load(cls, sid)   クラスメソッド: ディスクから JSON を読み、Session か None を返す
      compact()        セッション一覧用のメタデータのみの dict（messages なし）を返す

    インメモリキャッシュ:
      SESSIONS = {}    dict: session_id -> Session オブジェクト
      LOCK = threading.Lock()   定義済みだが現在 SESSIONS アクセス周りで未使用

    get_session(sid): SESSIONS キャッシュを確認、ミス時ディスクから読む、KeyError を投げる
    new_session(workspace, model): Session を作成、SESSIONS にキャッシュ、保存、返す
    all_sessions(): SESSION_DIR/*.json ＋ SESSIONS をスキャン、重複排除、updated_at でソート、
                    compact() dict のリストを返す

    all_sessions() は呼び出しのたびに完全なディレクトリスキャンを行います。
    10 セッションでは無視できる。1000+ では遅くなる。
    インデックスファイルによる修正は Architecture Phase C を参照。

title_from(): messages リストを取り、最初のユーザーメッセージを見つけ、先頭 64 文字を返す。run_conversation() 完了後に呼ばれ、セッションタイトルを事後的に設定します。

### 4.3 SSE ストリーミングエンジン

これはアーキテクチャ上最も興味深い部分です。2つのエンドポイントが協調します:

    POST /api/chat/start     ユーザーメッセージを受け取る。queue.Queue を作成し
                             STREAMS[stream_id] に格納、_run_agent_streaming() を実行する
                             デーモンスレッドを起動、即座に {stream_id} を返す。

    GET  /api/chat/stream    長寿命の SSE 接続。STREAMS[stream_id] から読み、
                             'done' または 'error' までイベントをブラウザへ転送。

キューレジストリ:

    STREAMS = {}               dict: stream_id -> queue.Queue
    STREAMS_LOCK = threading.Lock()

SSE イベントタイプとそのデータ形状:

    token       {"text": "..."}                         LLM トークンデルタ
    tool        {"name": "...", "preview": "..."}       ツール呼び出し開始
    approval    {"command": "...", "description": "...", "pattern_keys": [...]}
    done        {"session": {compact_fields + messages}} エージェント正常終了
    error       {"message": "...", "trace": "..."}       エージェントが例外を投げた

SSE ハンドラループ:
    - queue.get(timeout=30) でブロック
    - タイムアウト時（30秒イベントなし）: プロキシやファイアウォール越しに接続を保つため
      ハートビートコメント（": heartbeat\n\n"）を送る
    - 'done' または 'error' イベント時: ループを抜けて返す
    - BrokenPipeError と ConnectionResetError を黙ってキャッチ（ブラウザ切断）

ストリームクリーンアップ: _run_agent_streaming() は finally ブロックで自身の stream_id を STREAMS から pop します。ブラウザがストリーム途中で切断しても、デーモンスレッドは完了まで動き、その後クリーンアップします。キューが満杯になり put_nowait() 呼び出しは黙って失敗します（queue.Full はキャッチされる）。

フォールバック同期エンドポイント: POST /api/chat はまだ存在し、エージェントが終わるまで接続を開いたままにします。フロントエンドは使いませんが、デバッグに有用です。

### 4.4 エージェント呼び出し (_run_agent_streaming)

    def _run_agent_streaming(session_id, msg_text, model, workspace, stream_id):

1. SESSIONS からセッションを取得（ディスクからではない — セッションは /api/chat/start で更新済み）
2. TERMINAL_CWD、HERMES_EXEC_ASK、HERMES_SESSION_KEY の環境変数を設定
3. 次の設定で AIAgent を作成:
   - model=model、platform='cli'、quiet_mode=True
   - enabled_toolsets=CLI_TOOLSETS（config.yaml またはハードコードのデフォルト）
   - session_id=session_id
   - stream_delta_callback=on_token（トークンごとに発火）
   - tool_progress_callback=on_tool（ツール呼び出しごとに発火）
4. agent.run_conversation(user_message=msg_text, conversation_history=s.messages,
                          task_id=session_id) を呼ぶ
   注: キーワードは session_id ではなく task_id（よくある間違い、skill に文書化）
5. 返却時: s.messages を更新、title_from() を呼ぶ、セッション保存
6. ('done', {session: ...}) をキューに put
7. finally ブロック: 環境変数を復元、STREAMS から stream_id を pop

on_token コールバック:
    if text is None: return  # AIAgent からのストリーム終端センチネル
    put('token', {'text': text})

on_tool コールバック:
    put('tool', {'name': name, 'preview': preview})
    # 保留中の承認も即座に表面化:
    if has_pending(session_id):
        with _lock: p = dict(_pending.get(session_id, {}))
        if p: put('approval', p)

approval の on-tool 表面化ロジックにより、承認はツール発火直後（同じ SSE ストリーム内）に、次のポーリングサイクルを待たずに現れます。

### 4.5 承認システムの統合

承認システムは `tools/approval.py` の既存 Hermes gateway モジュールを使います。全状態はそのファイルのモジュールレベル変数に存在します:

    _pending = {}        dict: session_key -> pending_entry_dict
    _lock = Lock()       _pending を保護
    _permanent_approved  恒久承認された pattern key のセット

server.py がモジュール読み込み時に tools.approval をインポートし、すべてが同一プロセスで動くため、この状態は HTTP スレッドとエージェントデーモンスレッドの間で共有されます。

重要: これは Python のインポートがキャッシュされる（sys.modules）からこそ機能します。同じモジュールオブジェクトがどこでも使われます。もし approval モジュールがサブプロセスや importlib.reload() でインポートされたら、これは壊れます。

GET /api/approval/pending:
    - _pending[sid] を削除せず peek
    - {pending: entry} または {pending: null} を返す
    - S.busy が true の間ブラウザが 1500ms ごとに呼ぶ（ポーリングフォールバック）

POST /api/approval/respond:
    - _pending[sid] を pop（削除）
    - choice "once" または "session": 各キーに approve_session(sid, pattern_key) を呼ぶ
    - choice "always": approve_session ＋ approve_permanent ＋ save_permanent_allowlist を呼ぶ
    - choice "deny": pop のみ、何もしない（エージェントは拒否結果を得る）
    - {ok: true, choice: choice} を返す

### 4.6 ファイルアップロードパーサ

parse_multipart(rfile, content_type, content_length):
    - rfile から content_length バイトをすべてメモリに読む（MAX_UPLOAD_BYTES まで、デフォルト 20MB、HERMES_WEBUI_MAX_UPLOAD_MB で環境上書き可）
    - Content-Type ヘッダから boundary を抽出
    - 生バイトを b'--' + boundary で分割
    - 各パート: email.parser.HeaderParser で MIME ヘッダをパース
    - (fields, files) を返す。fields は {name: value}、files は {name: (filename, bytes)}

handle_upload(handler):
    - parse_multipart() を呼ぶ
    - 検証: file フィールド存在、filename 存在、セッション存在
    - filename をサニタイズ: 非単語文字を _ に置換、200 文字に切り詰め
    - バイトを session.workspace / safe_name に書く
    - {filename, path, size} を返す

なぜ cgi.FieldStorage を使わないか:
    - Python 3.11+ で非推奨
    - バイナリファイルで壊れる（黙って破損するか例外）
    - 手動パーサはすべてのファイルタイプを正しく扱う

### 4.7 ファイルシステム操作

safe_resolve(root, requested):
    - requested パスを root 相対で解決
    - .relative_to(root) を呼び、結果が root 内であることを表明
    - パストラバーサル（../../etc/passwd）で ValueError を投げる

list_dir(workspace, rel='.'):
    - safe_resolve を呼び、iterdir()
    - ソート: ディレクトリ優先、次にファイル、各グループ内で大小無視のアルファベット順
    - {name, path, type, size} 付きで最大 200 エントリを返す

read_file_content(workspace, rel):
    - safe_resolve を呼ぶ
    - MAX_FILE_BYTES = 200KB のサイズ上限を強制
    - errors='replace' で UTF-8 として読む（バイナリファイルは置換文字を表示）
    - {path, content, size, lines} を返す

---

## 5. フロントエンドアーキテクチャ: 現状

### 5.1 構造

フロントエンドは static/ から個別ファイルとして配信されます。1つの HTML テンプレート、1つの CSS ファイル、複数の JavaScript モジュール。外部依存には Prism.js（シンタックスハイライト）、Mermaid.js（図）、xterm.js、KaTeX アセットがあり、現在の static テンプレートの integrity/CSP 前提で読み込まれます。

アプリが読み込む主要 JS モジュール:
  1. ui.js         （約7216行）DOM ヘルパー、renderMd、ツールカード描画、グローバル状態
  2. workspace.js  （約369行）ファイルツリー、プレビュー、ファイル操作
  3. sessions.js  （約3517行）セッション CRUD、一覧描画、検索、SVG アイコン、ドロップダウンアクション、プロジェクトピッカー
  4. messages.js  （約2301行）send()、SSE イベントハンドラ、approval、トランスクリプト
  5. panels.js    （約6480行）Cron、skills、memory、workspace、profiles、todo、settings
  6. commands.js  （約1302行）スラッシュコマンドレジストリ、パーサ、オートコンプリートドロップダウン
  7. boot.js      （約1607行）イベント配線 ＋ ブート IIFE

sessions.js はモジュールレベルで `ICONS` 定数を定義し、全セッションアクションボタン（pin、unpin、folder、archive、unarchive、duplicate、trash）のハードコード SVG 文字列を持ちます。全アイコンは一貫したテーマのため `currentColor` を継承します。

3ペインレイアウト（static/index.html 内）:

    <aside class="sidebar">    左パネル: セッション一覧、ナビタブ、sidebar-footer の Hermes WebUI トリガー
    <main class="main">        中央: トップバー、メッセージエリア、承認カード、コンポーザ
    <aside class="rightpanel"> 右パネル: ワークスペースファイルツリーとファイルプレビュー

コンポーザフッターレイアウト（現状）:

    左クラスタ   attach ボタン、mic ボタン、会話ごとのモデルセレクタ
    右クラスタ   コンパクトな円形コンテキスト使用量バッジ、send ボタン

モデルセレクタは依然、新規セッション作成とセッション更新の権威あるコントロールです。グローバルなアプリ設定ではなくアクティブな会話にスコープされた感覚にするため、サイドバーから移されました。

### 5.2 グローバル状態

    const S = {
      session:      null,   // 現在の Session compact dict（model、workspace、title を含む）
      messages:     [],     // 現在のセッションの全 messages 配列
      entries:      [],     // 現在のディレクトリ一覧
      busy:         false,  // エージェント実行中は true（Send ボタンを無効化）
      pendingFiles: []      // 次のメッセージでアップロードするためにキューされた File オブジェクト
    }

    const INFLIGHT = {}
    // そのセッションでリクエストが進行中の間 session_id でキー付け
    // 値: {messages: [...snapshot...], uploaded: [...filenames...]}
    // 目的: リクエスト保留中にユーザーがセッションを切り替えても、
    //   戻ったとき保存状態ではなく進行中の状態を表示する

### 5.3 主要関数リファレンス

セッション管理:
    newSession()          POST /api/session/new、S.session を更新、localStorage に保存
    loadSession(sid)      GET /api/session?session_id=X、まず INFLIGHT を確認、S を更新
    deleteSession(sid)    POST /api/session/delete、active/inactive ケースを正しく処理
    renderSessionList()   GET /api/sessions、#sessionList DOM を再構築

チャット:
    send()                主アクション: ファイルアップロード、POST /api/chat/start、EventSource を開く
    uploadPendingFiles()  S.pendingFiles の各ファイルをアップロード、filenames 配列を返す
    appendThinking()      メッセージ一覧に三点アニメーションを追加
    removeThinking()      thinking ドットを除去（最初のトークンまたはエラー時に呼ばれる）

描画:
    renderMessages()      S.messages から #msgInner を完全再構築
    renderMd(raw)         自作 markdown レンダラ（既知の欠落は 5.4 参照）
    syncTopbar()          トップバーのタイトル、メタ、モデルチップ、ワークスペースチップを更新
    renderTray()          保留ファイルを示す attach トレイを更新

承認:
    showApprovalCard(p)   command/description テキスト付きで承認カードを表示
    hideApprovalCard()    承認カードを隠す、テキストをクリア
    respondApproval(ch)   POST /api/approval/respond、カードを隠す
    startApprovalPolling  setInterval 1500ms GET /api/approval/pending
    stopApprovalPolling   clearInterval

UI ヘルパー:
    setStatus(t)          フォールバックヘルパー: 非チャットのステータス/エラーメッセージをトースト表示
    setComposerStatus(t)  ターンスコープの状態用にインラインコンポーザステータスラベルを更新
    setBusy(v)            S.busy を設定、Send ボタンを無効/有効化、false でステータスをクリア
    showToast(msg, ms)    下部中央のフェードトースト（デフォルト 2800ms）
    showConfirmDialog(o)  共有のアプリ内確認モーダル、true/false を resolve
    showPromptDialog(o)   共有のアプリ内入力モーダル、string/null を resolve
    autoResize()          #msg テキストエリアを最大 200px まで自動リサイズ

ダイアログポリシー:
    ネイティブブラウザの confirm()/prompt() は Web UI で使いません。
    破壊的アクションは showConfirmDialog(...) を使い、成功時にトースト。
    軽量な命名フロー（新規ファイル/フォルダ/プロジェクト）は showPromptDialog(...) を使う。

ファイル:
    loadDir(path)         GET /api/list、#fileTree を再構築
    openFile(path)        GET /api/file、#previewArea に表示

トランスクリプト:
    transcript()          S.messages からダウンロード用の markdown 文字列を構築

ブート IIFE:
    localStorage キー 'hermes-webui-session' が最後の session_id を保存
    読み込み時: loadSession(saved) を試み、無いか失敗なら空状態にフォールバック
    ブート時にセッションを決して自動作成しない

### 5.4 Markdown レンダラ (renderMd)

HTML 安全性を備えた手書きの正規表現チェーン。次の順で処理します:

プリパス（v0.18.1）:
0a. フェンスドコードブロックとバッククォートスパンを退避（fence_stash 配列）
0b. 安全な HTML タグを markdown 等価物に変換:
    <strong>/<b> -> **text**、<em>/<i> -> *text*、<code> -> `text`、<br> -> 改行
0c. 退避したコードブロックを復元

パイプライン:
1. Mermaid ブロック（```mermaid ... ```）-> <div class="mermaid-block">
2. コードブロック（``` lang ... ```）-> 言語ヘッダ付き <pre><code>
3. インラインコード（`...`）-> <code>
4. 太字＋斜体（***..***）-> <strong><em>
5. 太字（**...**）-> <strong>
6. 斜体（*...*）-> <em>
7. 見出し（# ## ###）-> <h1> <h2> <h3>（内容に inlineMd() を使用）
8. 水平線（---+）-> <hr>
9. 引用（> ...）-> <blockquote>（内容に inlineMd() を使用）
10. 箇条書き（行頭の - か * か +）-> <ul><li>（inlineMd() を使用）
11. 番号付きリスト（行頭の N.）-> <ol><li>（inlineMd() を使用）
12. リンク（[text](https://...)）-> <a href target=_blank>
13. テーブル（| col | col |）-> <table>
14. セーフティネット: SAFE_TAGS 許可リストにない HTML タグを esc() でエスケープ
15. 段落ラップ: 残りの二重改行区切りブロック -> <p>

inlineMd() ヘルパー（v0.18.1）:
    リスト項目、引用、見出し内のインライン bold/italic/code/links を処理。SAFE_INLINE 許可リストで未知のタグをエスケープ。プリパス出力を二重エスケープしてしまう旧来の直接 esc() 呼び出しを置き換えた。

SAFE_TAGS 許可リスト:
    strong、em、code、pre、h1-6、ul、ol、li、table、thead、tbody、tr、th、td、hr、blockquote、p、br、a、div。それ以外はすべてエスケープ。

既知の欠落:
- 入れ子リスト: 単一正規表現パスで、多階層インデントは未処理
- 同一行での bold＋link 混在: 崩れた出力になり得る

### 5.5 モデルラベル解決（Sprint 1 で修正、コンポーザセレクタが再利用）

B3 は Sprint 1 で解決済み。現在のコードは MODEL_LABELS dict を使います:

    const MODEL_LABELS = {
      'openai/gpt-5.4-mini': 'GPT-5.4 Mini', 'openai/gpt-4o': 'GPT-4o',
      'openai/o3': 'o3', 'openai/o4-mini': 'o4-mini',
      'anthropic/claude-sonnet-4.6': 'Sonnet 4.6', 'anthropic/claude-sonnet-4-5': 'Sonnet 4.5',
      'anthropic/claude-haiku-3-5': 'Haiku 3.5', 'google/gemini-2.5-pro': 'Gemini 2.5 Pro',
      'deepseek/deepseek-chat-v3-0324': 'DeepSeek V3', 'meta-llama/llama-4-scout': 'Llama 4 Scout',
    };
    getModelLabel(m) => MODEL_LABELS[m] || (m.split('/').pop() || 'Unknown');

フォールバック: 未掲載のモデルは誤ったラベルではなく短い ID（最後の / 以降）を表示。新モデル追加: MODEL_LABELS にエントリを追加し、コンポーザフッターの <select> に <option> を追加。

### 5.6 セッション削除ルール（skill より）

これらのルールは重要です。GPT-5.4-mini が壊れた版を繰り返し再導入してきました。

1. deleteSession() は newSession() を決して呼ばない。削除は作成しない。
2. 削除セッションがアクティブで他のセッションが存在する場合: sessions[0]（最新）を読み込む。
3. 削除セッションがアクティブで残りセッションがない場合: 空状態を表示。
4. 削除セッションが非アクティブの場合: 一覧を再描画するだけ。
5. どの削除後も常に toast("Conversation deleted") を表示。

### 5.7 Send() セッションガード

send() 内の非同期操作の前に:
    const activeSid = S.session.session_id;

エージェント完了後:
    if (S.session && S.session.session_id === activeSid) {
      // 結果を適用、再描画
      setBusy(false);
    } else {
      // ユーザーが処理中にセッションを切り替えた
      // サイドバーのみ更新、新しいセッションで setBusy(false) を呼ばない
      await renderSessionList();
    }

これにより、処理中のセッション切り替えが、新しいセッションの状態を上書きしたり、誤ったセッションで Send ボタンをアンロックしたりするのを防ぎます。

---

## 6. データフロー: チャット往復の全体

メッセージを入力して Send を押したとき何が起きるかのステップごとのトレース:

1.  ユーザーが入力、Enter を押す。send() が呼ばれる。
2.  ガード: (!text && !pendingFiles) || S.busy なら return
3.  S.session が null の場合: await newSession()、await renderSessionList()
4.  activeSid = S.session.session_id をキャプチャ（あらゆる await の前に）
5.  uploadPendingFiles(): S.pendingFiles の各ファイルを /api/upload へ POST
    - アップロード進捗バーを表示
    - 完了時に S.pendingFiles をクリア
    - アップロード済み filenames の配列を返す
6.  text ＋ ファイルノートから msgText を構築
7.  userMsg {role:'user', content: displayText, attachments?: filenames} を構築
8.  userMsg を S.messages に push、renderMessages()、appendThinking() を呼ぶ
9.  setBusy(true)、setStatus('Hermes is thinking...')
10. INFLIGHT[activeSid] = {messages: [...S.messages], uploaded}
11. startApprovalPolling(activeSid)
12. POST /api/chat/start {session_id, message, model, workspace}
    サーバー: セッション保存、queue.Queue 作成、デーモンスレッド開始、{stream_id} を返す
13. ブラウザが EventSource('/api/chat/stream?stream_id=X') を開く
14. SSE ループ内:
    - 'token': assistantText += d.text、ensureAssistantRow()、markdown 描画
    - 'tool': setStatus('tool name...')
    - 'approval': showApprovalCard(d)
    - 'done': d.session から S を同期、renderMessages()、loadDir、renderSessionList、
               setBusy(false)、INFLIGHT[activeSid] を削除
    - 'error': エラーメッセージ表示、setBusy(false)
    - es.onerror: ネットワーク切断を処理（エラー表示、setBusy(false)）
15. 承認が必要な場合: ユーザーがボタンをクリック、respondApproval() が発火
    POST /api/approval/respond -> サーバーが _pending を pop、approve_* を呼ぶ
    エージェントがコマンドを再試行（is_approved() が True を返す）し継続

---

## 7. 依存マップ

server.py は api/ モジュール（config、helpers、models、workspace、upload、streaming）からインポートします。api/ モジュールはさらに Hermes 内部をインポートします:

    api/streaming.py のインポート:
      run_agent.AIAgent              主エージェントクラス。LLM ＋ ツール実行をラップ。
    api/config.py のインポート:
      yaml                           設定読み込み。
    server.py のインポート:
      tools.approval.*               モジュールレベルの承認状態（graceful フォールバック付き）。
    全モジュール共通の標準ライブラリ: json、os、re、sys、threading、time、traceback、
      uuid、http.server、pathlib、urllib.parse、email.parser、queue、collections

使用される AIAgent コンストラクタパラメータ:

    model=               OpenRouter モデル ID 文字列
    platform='cli'       ツール選択のためのプラットフォームコンテキストを設定
    quiet_mode=True      エージェント自身の stdout 出力を抑制
    enabled_toolsets=    config.yaml からのツールセット名のリスト
    session_id=          ツール状態キー付け（memory、todos など）に使用
    stream_delta_callback=   トークンデルタごとに呼ばれる（または None センチネル）
    tool_progress_callback=  ツール呼び出しごとに呼ばれる（name、preview、args）

AIAgent.run_conversation() パラメータ:

    user_message=           人間のターンのテキスト
    conversation_history=   先行メッセージリスト（OpenAI 形式）
    task_id=                セッション ID（注: session_id= ではなく task_id=）

戻り値:

    {
      'messages': [...],          新しいターンを含む完全な会話
      'final_response': '...',    最後のアシスタントテキスト応答
      'completed': True/False,    会話が正常に完了したか
      ...other fields
    }

---

## 8. 設定読み込み

起動時、server.py は ~/.hermes/config.yaml を読みます:

    cfg = yaml.safe_load(CONFIG_PATH.read_text())
    CLI_TOOLSETS = cfg.get('platform_toolsets', {}).get('cli', [...default...])

デフォルトツールセットリスト（ハードコードフォールバック）:
    browser、clarify、code_execution、cronjob、delegation、file、
    image_gen、memory、session_search、skills、terminal、todo、tts、vision、web

Web UI は常に完全な CLI ツールセットで動きます。UI からのセッションごとのツールセット制限はまだありません（計画は ROADMAP.md Wave 4 を参照）。

---

## 9. 既知のバグと技術的負債の要約

| ID  | 重大度 | 説明                                          | 状態           | 修正              |
|-----|----------|------------------------------------------------------|------------------|------------------|
| B1  | Critical | 承認の配線が未テスト。pattern_keys が表示されない     | FIXED Sprint 1   | カードがキーを表示。検証用に inject_test エンドポイント追加 |
| B2  | High     | file input に accept 属性がない                       | FIXED Sprint 1   | image/*、text/*、pdf、コード拡張子で accept= を追加 |
| B3  | High     | モデルチップのラベルが sonnet 部分文字列チェックをハードコード    | FIXED Sprint 1   | MODEL_LABELS マップ。短いモデル ID にフォールバック |
| B4  | High     | ストリーム途中の再読み込み: stream_id 喪失、再接続なし      | FIXED Sprint 1   | stream/status エンドポイント。localStorage 経由の再接続バナー |
| B5  | High     | INFLIGHT がメモリのみで、再読み込みで喪失              | FIXED Sprint 1   | localStorage の markInflight/clearInflight |
| B6  | Medium   | 新規セッションが常に DEFAULT_WORKSPACE を使う            | FIXED Sprint 3   | newSession() が S.session.workspace を /api/session/new に渡す |
| B7  | Medium   | サイドバータイトルのオーバーフロー: min-width:0 欠如          | FIXED Sprint 1   | .session-item に min-width:0 |
| B8  | Medium   | renderMd にテーブル・入れ子リストがない                 | PARTIAL Sprint 4 | テーブルは Sprint 2。入れ子リストは Sprint 4 で改善。完全修正は依然 Phase E |
| B9  | Medium   | 空のアシスタントメッセージが描画され得る                  | FIXED Sprint 1   | loadSession() が空テキストのアシスタントメッセージをフィルタ |
| B10 | Low      | ツール実行中も thinking ドットが残る               | FIXED Sprint 3   | 最初の tool イベントで removeThinking()。コンパクトな 'Running X...' 行を表示 |
| B11 | Low      | GET /api/session の ID なしが黙ってセッションを作成      | FIXED Sprint 1   | エラーメッセージ付き 400 を返す |
| B12 | Low      | プレビューパネルの display:none から flex へのレイアウトジャンプ      | FIXED Sprint 4   | visibility/opacity トランジションが display:none トグルを置換 |
| B13 | Low      | CORS ヘッダなし                                      | Open             | Phase H |
| B14 | Low      | 新規チャットのキーボードショートカットなし                    | FIXED Sprint 3   | Cmd/Ctrl+K がどこからでも newSession() をトリガー |
| TD1 | Critical | 環境変数がプロセスグローバル（並行リクエストのバグ） | PARTIAL Sprint 5 | スレッドローカル _set_thread_env() を追加。Sprint 4 のセッションごとロック。プロセスレベルの env はフォールバックとして依然書き込まれる。完全修正には terminal ツールがスレッドローカルを読む必要 |
| TD2 | High     | SESSIONS キャッシュ: エビクションなし、ロック欠如         | FIXED Sprint 5   | OrderedDict ＋ LRU 上限 100 ＋ アクセス時 move_to_end。LOCK は Sprint 1。完了 |
| TD3 | High     | テストカバレッジなし                                   | PARTIAL Sprint 1 | 19 の HTTP 統合テストを追加。ユニットテストは Phase A 分割待ち |
| TD4 | Medium   | 全コードが1ファイル（HTML/CSS/JS/Python 混在）    | FIXED Sprint 5   | Sprint 5 で JS を static/app.js に抽出（Sprint 9: app.js 削除、6モジュールに置換）。Phase A 完了 |
| TD5 | Medium   | リクエスト検証なし（KeyError -> 500 ＋ traceback）  | FIXED Sprint 4   | 全エンドポイントを堅牢化: /api/list、/api/file、/api/crons/* がクリーンな 400/404 を返す |
| TD6 | Low      | all_sessions() が呼び出しのたびに完全ディレクトリスキャン        | FIXED Sprint 5   | セッションインデックスファイル（_index.json）を save のたびに構築。all_sessions() がインデックスを O(1) で読む。Phase C 部分 |
| TD7 | Low      | 構造化ログなし                                | FIXED Sprint 1   | log_request() オーバーライドがリクエストごとに JSON を出力 |

---

## 10. アーキテクチャ改善ロードマップ

これらのフェーズは機能ロードマップと並行して進みます。各フェーズはソフトウェア品質を狙います: テスト容易性、回復力、保守性、モジュール性。

### Phase A: ファイル分離 -- COMPLETE

server.py を適切なパッケージに分割。Sprint 4〜10 で完了。

現在の構造:

    <repo>/
      server.py               エントリポイント ＋ HTTP Handler ディスパッチ（約446行）
      api/
        __init__.py
        routes.py             全 GET ＋ POST ルートハンドラ（約9772行）
        config.py             設定、定数、グローバル状態、モデル検出（約4139行）
        helpers.py            HTTP ヘルパー: j()、bad()、require()、safe_resolve()（約302行）
        models.py             Session モデル ＋ CRUD（約1927行）
        workspace.py          ファイル操作、ワークスペース管理（約810行）
        upload.py             Multipart パーサ、ファイルアップロードハンドラ（約284行）
        streaming.py          SSE エンジン、run_agent、cancel 対応（約4420行）
      static/
        index.html            HTML ドキュメント（ディスクから配信）
        style.css             全 CSS（約3767行）
        ui.js, workspace.js, sessions.js, messages.js, panels.js, commands.js, boot.js
      tests/
        conftest.py           隔離されたテストサーバー/状態フィクスチャ
        488 test files        5303 テスト収集
        test_regressions.py   恒久的な回帰ゲート

api/routes.py へのルート抽出は Sprint 11 で完了。server.py はアプリ全体に対して薄いシェルのまま: ヘッダ、構造化ログ、ルートへのディスパッチ、TLS ラッピング、main() を持つ Handler クラス。

### Phase B: スレッドセーフなリクエストコンテキスト（優先度: Critical、労力: Medium）

プロセスグローバルな環境変数を、スレッドローカルまたは明示的なパラメータ渡しに置き換える。

根本原因: TERMINAL_CWD、HERMES_EXEC_ASK、HERMES_SESSION_KEY は _run_agent_streaming() で os.environ 経由で設定される。2つの並行セッションが互いを上書きする。

修正オプション（推奨順）:

オプション1（最良）: AIAgent コンストラクタがコンテキスト dict を受け付けるか確認。workspace、exec_ask、session_key を直接渡す。サーバーコードでの環境変数使用ゼロ。

オプション2: threading.local() を使う:
    _ctx = threading.local()
    # _run_agent_streaming 内:
    _ctx.workspace = str(workspace)
    _ctx.session_key = session_id
    # 環境変数を読むツール内: まず _ctx を確認、os.environ にフォールバック

オプション3（暫定、単一ユーザーには安全）: 環境変数ブロックをセッションごとのロックでラップ:
    SESSION_AGENT_LOCKS = {}  # session_id -> Lock
    # セッションごとに同時1エージェント実行のみ
    with SESSION_AGENT_LOCKS.setdefault(session_id, threading.Lock()):
        os.environ[...] = ...
        result = agent.run_conversation(...)

Phase B はさらに: コードベースの他のすべての os.environ の読み書きを、同様のスレッド安全性問題について見直す。

### Phase C: セッションストアの改善 -- COMPLETE

3つの問題すべてを Sprint 5 で修正:

1. SESSIONS キャッシュ: LRU 上限 100 の OrderedDict、最古を自動エビクト。
2. LOCK: 全 SESSIONS dict の読み書きを LOCK でラップ（Sprint 1 から）。
3. セッションインデックス: `sessions/_index.json` を save/delete のたびに維持。
   `all_sessions()` が全 JSON をスキャンする代わりにインデックスファイルを読む（O(1)）。

### Phase D: 入力検証とエラー処理 -- COMPLETE

Sprint 4〜6 で完了:

1. パラメータ検証用の `api/helpers.py` の `require()` と `bad()` ヘルパー。
2. 全エンドポイントが traceback ではなくクリーンな 400/404 を返す。
3. `log_request()` オーバーライド経由の構造化 JSON リクエストログ（Sprint 1）。

### Phase E: フロントエンドのモジュール化 -- COMPLETE

Sprint 5、6、9 で完了:

1. HTML を `static/index.html` に抽出（Sprint 6）。
2. CSS を `static/style.css` に抽出（Sprint 4）。
3. `app.js` を Sprint 9 で削除、6つの焦点を絞ったモジュールに置換:
   `ui.js`、`workspace.js`、`sessions.js`、`messages.js`、`panels.js`、`boot.js`。
   依存順で標準の `<script>` タグ（ES モジュールではない）として読み込み。
4. シンタックスハイライト用に Prism.js を CDN 経由で追加（Sprint 8）、遅延読み込み。

残り: renderMd() は依然手書きの正規表現チェーン。テーブルは部分対応。marked.js ＋ DOMPurify への置き換えは将来の改善（ブロッキングではない）。

### Phase F: API 設計のクリーンアップ（優先度: Low、労力: Medium）

1. バージョンプレフィックス: 全新規エンドポイントに /api/v1/ を追加。
   後方互換のため /api/* をエイリアスとして残す。

2. 標準レスポンスエンベロープ:
   成功: {"ok": true, "data": {...}}
   エラー: {"ok": false, "error": "message", "code": "ERROR_CODE"}

3. セッション一覧のページネーション:
   GET /api/v1/sessions?limit=30&offset=0
   レスポンス: {"ok": true, "data": {"sessions": [...], "total": N, "has_more": false}}

4. 一貫した命名: 全 JSON キーに snake_case を使う。

### Phase G: 可観測性 -- MOSTLY COMPLETE

1. 構造化 JSON ログ: COMPLETE（Sprint 1）。リクエストごとの JSON がアクティブなランチャーログに出力される（`start.sh` は `~/.hermes/webui/bootstrap-8787.log`、`ctl.sh` は `~/.hermes/webui.log`）。
2. 拡張 /health: COMPLETE（Sprint 7）。`active_streams`、`uptime_seconds` を返す。
3. GET /api/debug/stats: 未実装。低優先度。

### Phase H: 認証（優先度: Low、労力: Medium）

非 SSH トンネルデプロイ向けの任意のパスワードゲート。

1. HERMES_WEBUI_PASSWORD 環境変数が認証を有効化
2. ログインページ: 最小限のダークフォーム、POST /api/auth/login
3. ログイン成功時にサーバーが HttpOnly ＋ SameSite=Strict Cookie を設定
4. HERMES_WEBUI_PASSWORD 設定時、全 API エンドポイントが Cookie を確認
5. Cookie 有効期間: 最終アクティビティから 30 日

### Phase I: テスト基盤 -- COMPLETE

488 テストファイルにわたる 5303 テスト ＋ 回帰ゲート。pytest フィクスチャは、`HERMES_WEBUI_TEST_PORT` / `HERMES_WEBUI_TEST_STATE_DIR` が明示的に固定しない限り、リポジトリパスから隔離ポートと状態ディレクトリを導出。本番データには決して触れない。

`conftest.py` のフィクスチャ: 自動クリーンアップ、profile/config 隔離、cron 隔離、ワークスペースリセット、テストサーバーのライフサイクル。

### Phase J: パフォーマンス（優先度: Low、労力: High）

単一ユーザーのカジュアル利用を超えるスケール向け。

1. セッションインデックス（Phase C 前提）: O(1) のセッション一覧読み込み
2. メッセージページネーション: /api/session が直近 50 メッセージを返し、古いものはページング
3. フロントエンドの仮想スクロール: メッセージ一覧とセッション一覧の両方に IntersectionObserver
4. ストリームクリーンアップのバックグラウンドスレッド: 5 分より古い STREAMS エントリをエビクト
5. ファイルツリーの遅延読み込み: クリックで展開しサブディレクトリ内容を取得

---

## 11. 新しい API エンドポイントの追加方法

このとおりのパターンに従ってください。参考に do_GET/do_POST の既存ハンドラを確認してください。

### バックエンド（server.py -> 将来: api/handlers.py）

GET エンドポイント:

    # do_GET 内、404 フォールバック行の前:
    if parsed.path == '/api/your/endpoint':
        qs = parse_qs(parsed.query)
        param = qs.get('param', [''])[0]
        if not param:
            return j(self, {'error': 'param is required'}, status=400)
        # 作業
        return j(self, {'result': value})

POST エンドポイント（/api/upload チェックの後、ボディはパース済み）:

    if parsed.path == '/api/your/endpoint':
        value = body.get('field', '')
        if not value:
            return j(self, {'error': 'field is required'}, status=400)
        # 作業
        return j(self, {'ok': True, 'data': result})

有効なセッションを要するエンドポイント:

    sid = body.get('session_id', '')
    try:
        s = get_session(sid)
    except KeyError:
        return j(self, {'error': 'Session not found'}, status=404)

Hermes Python モジュールを呼ぶエンドポイント:

    # 例: cron.jobs を呼ぶ
    import sys
    sys.path.insert(0, str(Path(__file__).parent.parent))
    from cron.jobs import list_jobs
    jobs = list_jobs(include_disabled=True)
    return j(self, {'jobs': jobs})

### フロントエンド（6つの static JS モジュール: ui.js、workspace.js、sessions.js、messages.js、panels.js、boot.js）

シンプルな GET fetch:

    const data = await api('/api/your/endpoint?param=' + encodeURIComponent(value));
    // data はパース済み JSON レスポンス、エラー時に throw

POST:

    const data = await api('/api/your/endpoint', {
      method: 'POST',
      body: JSON.stringify({field: value})
    });

api() ヘルパー:

    async function api(path, opts={}) {
      const r = await fetch(path, {headers:{'Content-Type':'application/json'},...opts});
      const d = await r.json();
      if (!r.ok) throw new Error(d.error || r.statusText);
      return d;
    }

---

## 12. よく使うデバッグコマンド

    # サーバーヘルスとセッション数
    curl -s http://127.0.0.1:8787/health | python3 -m json.tool

    # サーバーログをライブで tail
    tail -f ~/.hermes/webui/bootstrap-8787.log
    tail -f ~/.hermes/webui.log  # ctl.sh で起動した場合

    # 全セッションを一覧（メタデータのみ）
    curl -s http://127.0.0.1:8787/api/sessions | python3 -m json.tool

    # メッセージ込みで完全なセッションを調べる
    SID=your_session_id_here
    curl -s "http://127.0.0.1:8787/api/session?session_id=$SID" | python3 -m json.tool

    # サーバーをクリーンに kill して再起動
    pkill -f "python.*server.py"
    <repo>/start.sh

    # サーバープロセスが動いているか確認
    ps aux | grep "server.py"

    # ディスク上のセッションファイルを調べる
    ls -lt ~/.hermes/webui/sessions/
    cat ~/.hermes/webui/sessions/SESSION_ID.json | python3 -m json.tool

    # セッション内のメッセージ数を数える
    python3 -c "import json; d=json.load(open('sessions/SID.json')); print(len(d['messages']))"

    # 承認モジュールの状態を確認
    cd <agent-dir>
    venv/bin/python -c "from tools.approval import _pending; print(_pending)"

    # アクティブな SSE ストリームを確認（サーバーアクセスが必要）
    curl -s http://127.0.0.1:8787/health  # ストリームは未公開、Phase G で追加

    # メッセージを持つ全セッションを探す（Untitled の空は除く）
    ls ~/.hermes/webui/sessions/ | xargs -I{} python3 -c "
    import json, sys
    d = json.load(open('~/.hermes/webui/sessions/{}'))
    if d['messages']: print('{}', d['title'][:50])
    " 2>/dev/null

---

## 13. アーキテクチャ決定記録（ADR）

### ADR-001: 単一ファイルサーバー
決定: 全コードを server.py に。
理由: ビルドステップなし、容易なエージェント変更、デプロイの複雑さゼロ。
トレードオフ: ファイルサイズとともに保守負担が増える。
解決: Phase A がファイルを分割。

### ADR-002: HTML を Python 生文字列として
決定: フロントエンドを server.py に r"""..." として埋め込む。
理由: 静的ファイルサーバーやビルドシステムなしでフロントエンドを配信する最も単純な方法。
トレードオフ: エディタのシンタックスハイライトなし、複雑なパッチ、大きな編集での base64 の曲芸。
解決: Phase A がディスクから配信される static/index.html に移す。

### ADR-003: ThreadingHTTPServer
決定: Python 標準ライブラリ、asyncio ではなく同期スレッド。
理由: 依存なし、同期エージェント呼び出しがスレッドに自然に収まる。
トレードオフ: 並行ユーザー数に線形にメモリがスケール。スレッドプールは無制限。
解決: 単一ユーザーには許容。必要なら Phase J が並行制限を追加。

### ADR-004: WebSocket ではなく SSE
決定: ストリーミングに Server-Sent Events。
理由: WebSocket より単純、単方向、アップグレードハンドシェイクなし、EventSource は標準ブラウザ API。
トレードオフ: サーバー→クライアントのみ。承認イベントはエージェントスレッドからの SSE ＋ ポーリングフォールバックを使う。
解決: 切り替え予定なし。SSE で十分。

### ADR-005: モジュールレベルの承認状態
決定: tools/approval.py が全スレッド共有のモジュールレベル _pending dict を使う。
理由: 承認システムは既存。同一 Python プロセス経由の状態共有が機能する。
トレードオフ: マルチプロセス（gunicorn ワーカー）やサブプロセスへ移ると壊れる。
解決: 制約を文書化。スケールが必要になれば SQLite へ移す。

### ADR-006: 認証なし
決定: 当初は認証なし。
理由: SSH トンネル経由の localhost のみ。トランスポート層（SSH）が既に認証済みのとき、認証はセキュリティ便益なしに複雑さを加える。
トレードオフ: localhost アクセスを持つ VPS 上の誰でもサーバーを使える。
解決: Phase H が直接アクセスデプロイ向けに任意のパスワードゲートを追加。

### ADR-007: 環境変数経由の承認状態
決定: HERMES_EXEC_ASK と HERMES_SESSION_KEY を os.environ 経由で渡す。
理由: tools/approval.py と terminal_tool.py が既にこれらの環境変数を読む。
トレードオフ: プロセスグローバル。2つの並行チャットリクエストが互いを上書き。
解決: Phase B がスレッドローカルまたは明示的なパラメータ渡しに置き換える。

---

## 14. バージョン履歴

    v0.1  初期 MVP: 単一ファイルサーバー、同期 /api/chat、ストリーミングなし
    v0.2  /api/chat/start ＋ /api/chat/stream 経由の SSE ストリーミング
    v0.2  INFLIGHT セッションガード、セッション削除ルール、トースト UI
    v0.2  バイナリファイルアップロード修正（cgi.FieldStorage を parse_multipart に置換）
    v0.2  承認カード UI を tools/approval.py に配線
    v0.2  承認 SSE イベント（ツール呼び出し時に即座に表面化）
    v0.3  Sprint 1（2026年3月30日）:
            バグ修正: B1 B2 B3 B4/B5 B7 B9 B11 すべて解決
            アーキテクチャ: SESSIONS の LOCK、セクションヘッダ、構造化 JSON ログ
            テスト: 19/19 HTTP 統合テスト合格
            機能: プロバイダーグループ付き10モデルドロップダウン、再接続バナー、
                       GET /api/chat/stream/status、GET /api/approval/inject_test
    v0.4  Sprint 2（2026年3月30日）:
            機能: /api/file/raw 経由の画像プレビュー、右パネルの描画 markdown、
                       renderMd() のテーブル対応、スマートファイルアイコン、パスバーのタイプバッジ
            テスト: 8 新規、合計 27/27 合格
    v0.5  [計画] Wave 1 機能: cron ビューア、skills ビューア、memory ビューア
    v0.5  Sprint 3（2026年3月30日）:
            機能: サイドバーナビタブ（Chat/Tasks/Skills/Memory）、cron ビューア、
                       skills ビューア（検索 ＋ SKILL.md プレビュー）、memory ビューア
            バグ修正: B6、B10、B14
            アーキ: Phase D 部分（require()/bad() 検証ヘルパー）
            新規エンドポイント: /api/crons、/api/crons/output、/api/crons/run、/api/crons/pause、
                           /api/crons/resume、/api/skills、/api/skills/content、/api/memory
            テスト: 21 新規、合計 48/48
    v0.6  Sprint 4（2026年3月30日）:
            移転: ソースを <repo>/ へ移動、シンボリックリンクで戻す
            Phase A 部分: CSS を static/style.css に抽出、ディスクから配信
            Phase B 部分: セッションごとエージェントロック（SESSION_AGENT_LOCKS）
            機能: セッションリネーム（インライン）、セッション検索、ファイル削除、ファイル作成
            バグ修正: B12、B8 改善、TD5 完了
            新規エンドポイント: /api/session/rename、/api/sessions/search、/api/file/delete、/api/file/create、GET /static/*
            テスト: 20 新規、合計 68/68
    v0.7  Sprint 5（2026年3月30日）:
            アーキ: Phase A 完了（JS -> static/app.js）、TD2 LRU キャッシュ、TD1 スレッドローカル、Phase C インデックス
            機能: ワークスペース管理パネル ＋ トップバークイック切り替え、メッセージコピー、インラインファイルエディタ
            新規エンドポイント: /api/workspaces、/api/workspaces/add、/api/workspaces/remove、/api/workspaces/rename、/api/file/save
            新規状態ファイル: workspaces.json、last_workspace.txt、sessions/_index.json
            テスト: 18 新規、合計 86/86
    v0.8  Sprint 6（2026年3月31日）:
            Phase E 完了: HTML を static/index.html へ（server.py は今や903行、純 Python）
            Phase D 完了: 全エンドポイント検証済み
            機能: リサイズ可能パネル（localStorage）、UI からの cron 作成、セッション JSON エクスポート
            バグ修正: ファイルエディタからの Escape が編集をキャンセル
            新規エンドポイント: POST /api/crons/create、GET /api/session/export
            テスト: 16 新規、合計 106/106
    v0.10  Sprint 8（2026年3月31日）:
            機能: メッセージの編集＋再生成、直前応答の再生成、会話クリア、
                       Prism.js シンタックスハイライト、メッセージキュー（MSG_QUEUE ＋ アイドル時 drain）、
                       INFLIGHT 優先 loadSession（切替で離れて戻ってもメッセージ保持）
            バグ修正: A1（再接続バナー誤検知）、A2（セッション一覧スクロールクリップ）
            新規エンドポイント: POST /api/session/clear、POST /api/session/truncate
            テスト: 14 新規、合計 139/139
            JS: MSG_QUEUE グローバル、updateQueueBadge()、setBusy drain ロジック、busy 時 send() がキュー、
                loadSession がサーバー fetch 前に INFLIGHT を確認
    v0.12.2 並行性スイープ（2026年3月31日）:
            R10-R15: 承認のクロスセッション、セッションごとアクティビティバー、切替復帰時のライブカード
            復元、done 後の settled カード、モデルソース、newSession カードクリア。190/190 テスト。
    v0.12  Sprint 10（2026年3月31日）:
            アーキ: server.py を api/ モジュールに分割（config、helpers、models、workspace、upload、streaming）
            機能: バックグラウンドタスクキャンセル、cron 実行履歴、ツールカード UX 磨き込み
            スプリント後修正: SSE cancel イベントがループを抜ける、setBusy(false) で Cancel ボタン常時非表示、
              S.activeStreamId 初期化、ツールカード show-more が data 属性を使用、バージョンラベル v0.12、
              Session.__init__ **kwargs 前方互換、HERMES_HOME 経由のテスト cron 隔離、
              テスト間で conftest が last_workspace リセット、アシスタントターンでツールカードをグループ化
            テスト: 18 新規、合計 167/167
            修正された回帰: uuid、AIAgent、has_pending、SSE cancel ループ、Session.__init__ tool_calls
            test_regressions.py: 10 テスト — 導入バグごとに1つ、恒久回帰ゲート
            修正後合計: 177/177
    v0.11  Sprint 9（2026年3月31日）:
            アーキ: app.js 削除。ui.js、workspace.js、sessions.js、messages.js、panels.js、boot.js に置換
            機能: ツール呼び出しカード（インライン折りたたみ、ライブ＋履歴）、添付の永続化、
                       todo リストパネル（セッション履歴からツール結果をパース）
            テスト: 10 新規、合計 149/149
    v0.9  Sprint 7（2026年3月31日）:
            機能: cron 編集＋削除、skill 作成/編集/削除、memory 書き込み、セッション内容検索
            アーキ: Phase G 部分（/health に active_streams＋uptime）、git init
            バグ修正: A1（アクティビティバー min-height）、A2（モデルチップ同期）、A3（cron 出力オーバーフロー）
            新規エンドポイント: /api/crons/update、/api/crons/delete、/api/skills/save、/api/skills/delete、
                           /api/memory/write、/api/sessions/search（拡張）
            テスト: 19 新規、合計 125/125


---

## 15. スプリントログ

このセクションは各スプリントで実際に何を作り変更したかを記録します。コードベースの恒久的な歴史です。各スプリント終了時に更新してください。

### Sprint 1（2026年3月30日）: バグ修正、アーキ基盤、最初のテスト

**トラック:** バグ修正（7）、アーキテクチャ（3）、テスト（1）
**テスト結果:** 19/19 合格
**バックアップ:** server.py.sprint1.bak

#### 適用したバグ修正

| ID  | 説明                          | 変更                                                                 |
|-----|--------------------------------------|------------------------------------------------------------------------|
| B3  | 新モデルでモデルチップラベルが誤り | 部分文字列チェックを MODEL_LABELS dict に置換。10モデル対応  |
| B7  | サイドバータイトルのオーバーフロー                | .session-item に min-width:0 を追加                                     |
| B11 | /api/session GET が黙ってセッション作成 | session_id 欠如時にエラーメッセージ付き 400 を返す          |
| B2  | file input に accept 属性なし        | image/*、text/*、pdf、json、一般的なコード拡張子で accept= を追加  |
| B9  | 空のアシスタントメッセージが描画       | loadSession() が描画前に空テキストのアシスタントメッセージをフィルタ  |
| B1  | 承認カードに pattern コンテキストなし | showApprovalCard() が pattern_keys を description テキストに付加        |
| B4/B5 | ストリーム途中の再読み込みでコンテキスト喪失 | localStorage の markInflight/clearInflight。checkInflightOnBoot() が金色再接続バナーを表示。GET /api/chat/stream/status エンドポイント追加 |

モデルドロップダウンも 2 から 10 オプションに拡張、<optgroup> でプロバイダー別にグループ化。

#### 適用したアーキテクチャ改善

| 項目   | 説明                          | 変更                                                                    |
|--------|--------------------------------------|---------------------------------------------------------------------------|
| Arch-1 | セクションヘッダ                      | server.py を論理ゾーンに分ける 8 個の明確な # === SECTION === バナー   |
| Arch-2 | SESSIONS dict 周りの LOCK            | get_session、new_session、delete が LOCK を保持。レースコンディションを排除 |
| Arch-3 | 構造化リクエストログ           | log_request() オーバーライドがリクエストごとに /tmp/webui-mvp.log へ JSON を出力      |

リクエストログ形式:
    {"ts": "2026-03-30T17:30:08Z", "method": "GET", "path": "/health", "status": 200, "ms": 0.1}

#### 追加したテストスイート

ファイル: webui-mvp/tests/test_sprint1.py（19 テスト）
ファイル: webui-mvp/tests/__init__.py

テストカテゴリ:
    ヘルスチェック（1）
    セッション CRUD: 作成、読み込み、更新、削除、ソート、B11 落とし穴（6）
    Multipart パーサのユニットテスト: テキストファイル、バイナリ/PNG（2）
    HTTP アップロード: 成功、大きすぎ、ファイルなし、不正なセッション（4）
    承認 API: pending/なし、inject+deny、inject+session-approve（3）
    ストリームステータスエンドポイント（1）
    ファイルブラウザ: ディレクトリ一覧、パストラバーサルブロック（2）

テスト実行:
    cd <agent-dir>
    venv/bin/python -m pytest webui-mvp/tests/test_sprint1.py -v

#### セクション 5.5 更新（B3 解決）

モデルチップラベルのバグは修正済み。syncTopbar() の MODEL_LABELS オブジェクト:

    const MODEL_LABELS = {
      'openai/gpt-5.4-mini':             'GPT-5.4 Mini',
      'openai/gpt-4o':                   'GPT-4o',
      'openai/o3':                       'o3',
      'openai/o4-mini':                  'o4-mini',
      'anthropic/claude-sonnet-4.6':     'Sonnet 4.6',
      'anthropic/claude-sonnet-4-5':     'Sonnet 4.5',
      'anthropic/claude-haiku-3-5':      'Haiku 3.5',
      'google/gemini-2.5-pro':           'Gemini 2.5 Pro',
      'deepseek/deepseek-chat-v3-0324':  'DeepSeek V3',
      'meta-llama/llama-4-scout':        'Llama 4 Scout',
    };
    getModelLabel(m) => MODEL_LABELS[m] || (m.split('/').pop() || 'Unknown');

フォールバック: '/' で分割し最後のセグメントを使うため、未掲載のモデルは誤ったハードコードラベルではなく短い識別子を表示。

#### バージョン履歴更新

    v0.3  Sprint 1: B3/B7/B11/B2/B9/B1/B4/B5 バグ修正
    v0.3  Sprint 1: モデルドロップダウンをプロバイダーグループで10モデルに拡張
    v0.3  Sprint 1: SESSIONS dict 周りに LOCK 追加（スレッド安全性）
    v0.3  Sprint 1: server.py 全体にセクションヘッダ追加
    v0.3  Sprint 1: log_request() オーバーライド経由の構造化 JSON リクエストログ
    v0.3  Sprint 1: GET /api/chat/stream/status エンドポイント
    v0.3  Sprint 1: 再接続バナー（markInflight/clearInflight/checkInflightOnBoot）
    v0.3  Sprint 1: GET /api/approval/inject_test エンドポイント（テスト専用）
    v0.3  Sprint 1: 最初の pytest スイート、19 テスト、全合格

---

## 16. アーキテクチャフェーズ優先度マトリクス

アーキテクチャ作業の優先順位付け用クイックリファレンス表。フェーズはセクション 10 から。

| Phase | 名前                        | 優先度 | 労力 | ブロック         | 状態     |
|-------|-----------------------------|----------|--------|----------------|------------|
| A+E   | ファイル分離 ＋ フロントエンド  | High     | Medium | F              | COMPLETE Sprint 6+9（HTML->index.html、JS->6 モジュール、app.js 削除。server.py は純 Python 約1150行） |
| B     | スレッドセーフなリクエストコンテキスト | Critical | Medium | なし        | PARTIAL（Sprint 4: セッションごとロック追加。グローバル環境変数は依然使用） |
| C     | セッションストアの改善  | Medium   | Medium | J              | PARTIAL Sprint 5（インデックスファイル ＋ LRU キャッシュ。LRU エビクションポリシーとページネーションは未） |
| D     | 入力検証            | Medium   | Low    | なし        | COMPLETE Sprint 6（approval/respond ＋ file/raw 堅牢化。全エンドポイント検証済み） |
| E     | フロントエンドのモジュール化     | Medium   | High   | A 必須     | Pending    |
| F     | API 設計のクリーンアップ          | Low      | Medium | A 必須     | Pending    |
| G     | 可観測性               | Low      | Low    | なし        | Partial（Sprint 7: /health に active_streams＋uptime 追加。ログローテーションは未） |
| H     | 認証              | Low      | Medium | なし        | Pending    |
| I     | テスト基盤         | High     | High   | A,D 必須   | Partial(*) |
| J     | パフォーマンス                 | Low      | High   | C 必須     | Pending    |

(*) Phase G は部分: 構造化リクエストログは Sprint 1 で完了。完全な可観測性（health 詳細、debug/stats エンドポイント、ログローテーション）は残る。
(*) Phase I は部分: HTTP 統合テストスイートは Sprint 1 で開始。隔離モジュールのユニットテストはまず Phase A のファイル分割を要する。

推奨実行順:
    1. Phase B（スレッド安全性）: クリティカル、低リスク、ファイル変更不要
    2. Phase D（入力検証）: 低労力、エラーメッセージを即改善
    3. Phase A（ファイル分割）: E、F、完全な Phase I を可能にする
    4. Phase G 残り（health 詳細、debug エンドポイント）: 1〜2 時間
    5. Phase C（セッションインデックス）: セッション数増加に伴い必要
    6. Phase E（フロントエンドモジュール ＋ marked.js）: 最大の UX 改善
    7. Phase I（完全テストスイート）: A がインポート可能モジュールを与えた後
    8. Phase F、H、J: 低優先度、必要時に着手

---

## 17. エージェントコントリビューター向けの作業規約

このセクションは、特にこのコードベースで作業するエージェント（Hermes インスタンス、サブエージェント、Codex など）向けです。ファイルに触れる前に読んでください。

### 変更を加える前に

1. 本ドキュメント（ARCHITECTURE.md）を完全に読む。特にセクション 4、5、ADR。
2. `api/` または `static/` 配下の該当モジュールを調べる。`server.py` はルーティングシェルに過ぎない。
3. 最近何が変わったか理解するためスプリントログ（セクション 15）を確認。
4. ベースライン確認のためまず該当テストスライスを実行、例:
   venv/bin/python -m pytest tests/test_regressions.py -q
5. サーバーヘルス確認: curl -s http://127.0.0.1:8787/health

### 変更を加える

編集はその挙動を所有するモジュールにスコープを保つ。機械的なパッチでは正確な文字列マッチを使い、置換前に意図した旧文字列が見つかったことを検証する。

あらゆる変更後:
    venv/bin/python -m py_compile server.py             # 構文チェック
    curl -s http://127.0.0.1:8787/health                # サーバーが生きている
    venv/bin/python -m pytest tests/ -v                 # テストが依然合格

### クリティカルルール（これらを退行させない）

これらのパターンは何度も壊され修正されてきました。再導入しないでください。

RULE-1: deleteSession() は newSession() を決して呼んではならない。
    削除は作成しない。削除セッションがアクティブで他が残るなら sessions[0] を読み込む。残らないなら空状態を表示。セクション 5.6 を参照。

RULE-2: do_POST で /api/upload を read_body() の前に確認する。
    read_body() はリクエストボディを消費する。アップロードパースもボディを要する。順序が重要。セクション 4.1 を参照。

RULE-3: run_conversation() は session_id= ではなく task_id= を取る。
    task_id が正しいキーワード引数。session_id= は黙って TypeError を投げる。

RULE-4: stream_delta_callback はストリーム終端センチネルとして None を受け取る。
    on_token コールバックはガードが必要: if text is None: return

RULE-5: send() はあらゆる await の前に activeSid をキャプチャする。
    await 保留中にアクティブセッションが変わり得る。最初にキャプチャし、返却時にガード。

RULE-6: ブート IIFE はセッションを決して自動作成しない。
    セッションを作るのは 2 箇所のみ: + ボタンと、S.session が null のときの send()。

RULE-7: 全 SESSIONS dict アクセスは LOCK を保持する。
    LOCK はモジュールレベルの threading.Lock()。使い方: with LOCK: ...

RULE-8: API クライアントに traceback を露出しない。
    500 レスポンスは完全な traceback ではなく {"error": "Internal server error"} を返すべき。
    （現状 traceback が露出。Phase D で修正。悪化させないこと。）

RULE-9: マルチパターン承認には pattern_key ではなく pattern_keys。
    承認モジュールは pattern_key（単数、レガシー）と pattern_keys（複数、全マッチパターン）の両方を含み得る。承認時は常に pattern_keys を反復。

### 新しい API エンドポイントの追加

正確なコードパターンはセクション 11 を参照。要約:
- GET: do_GET の 404 フォールバックの前に追加
- POST: /api/upload チェックの後、read_body() の後、do_POST の 404 フォールバックの前に追加
- 必須フィールドを常に検証、欠如/不正な入力には 400 を返す
- 常に get_session(sid) を try/except KeyError -> 400 か 404 で使う
- test_sprint1.py か新しいテストファイルにテストを追加

### 本ドキュメントの更新

次のときに ARCHITECTURE.md を更新する:
- セクション 9 のバグを修正したとき（行を更新、解決済みにマーク）
- アーキテクチャフェーズを完了したとき（セクション 16 のマトリクスを更新）
- 新しいエンドポイントを追加したとき（セクション 4.1 のルーティング表に追加）
- 新しい落とし穴やルールを発見したとき（セクション 17 に追加）
- スプリントを完了したとき（セクション 15 に新エントリを追加）

本ドキュメントはコードベースの記憶です。更新されなければ、将来のエージェントは同じ間違いを繰り返します。

---

## 18. エンドポイントリファレンス（現状）

Sprint 1（v0.3）時点の全 HTTP エンドポイントの完全リスト。

### GET エンドポイント

    /                          完全な HTML アプリを返す（index ページ）
    /index.html                / と同じ
    /health                    {"status":"ok","sessions":N}
    /api/session               ?session_id=X -> 完全なセッション ＋ messages。ID なしで 400。
    /api/sessions              全セッション compact() dict の一覧、updated_at でソート
    /api/list                  ?session_id=X&path=. -> セッションワークスペースのディレクトリ一覧
    /api/file                  ?session_id=X&path=rel -> ファイル内容（テキスト、200KB 上限）
    /api/chat/stream           ?stream_id=X -> SSE ストリーム。長寿命。token/tool/
                               approval/done/error イベントを発行。
    /api/chat/stream/status    ?stream_id=X -> {"active": true/false, "stream_id": X}
    /api/approval/pending      ?session_id=X -> {"pending": entry_or_null}
    /api/approval/inject_test  ?session_id=X&pattern_key=K&command=C -> テスト専用エンドポイント。
                               保留承認エントリをサーバープロセスに注入。
    /api/file/raw              ?session_id=X&path=P -> 正しい MIME タイプの生ファイルバイト。
                               画像プレビューに使用。safe_resolve でパストラバーサル保護。
                               ファイルが無ければ 404 JSON を返す。

### POST エンドポイント

    /api/upload                multipart/form-data。フィールド: session_id、file。filename を返す。
    /api/session/new           {"model"?, "workspace"?} -> 新規セッション
    /api/session/update        {"session_id", "workspace"?, "model"?} -> 更新されたセッション
    /api/session/delete        {"session_id"} -> {"ok": true}
    /api/chat/start            {"session_id", "message", "model"?, "workspace"?}
                               -> {"stream_id", "session_id"}。エージェントデーモンスレッドを開始。
    /api/chat                  （フォールバック、同期）{"session_id", "message", "model"?, "workspace"?}
                               -> エージェント終了までブロック。完全な結果を返す。
    /api/approval/respond      {"session_id", "choice": once|session|always|deny}
                               -> {"ok": true, "choice": choice}

### Sprint 3 で追加された GET エンドポイント

    /api/crons                 全 cron ジョブ。{jobs: [...]} を返す。
    /api/crons/output          ?job_id=X&limit=N -> {outputs: [{filename, content}]}
    /api/skills                全 skills。{skills: [{name, description, category}]} を返す
    /api/skills/content        ?name=X -> SKILL.md 内容を含む完全な skill データ
    /api/memory                MEMORY.md ＋ USER.md ＋ SOUL.md。{memory, user, soul, *_path, *_mtime} を返す

### Sprint 3 で追加された POST エンドポイント

    /api/crons/run             {job_id} -> デーモンスレッドで実行をトリガー。{ok, status} を返す。
    /api/crons/pause           {job_id} -> {ok, job} または 404。
    /api/crons/resume          {job_id} -> {ok, job} または 404。

---

## Sprint 2 ログエントリ（2026年3月30日）

セクション 15 スプリントログに追加。

### Sprint 2: リッチファイルプレビュー（2026年3月30日）

**トラック:** 機能（4 サブ機能）、テスト（8 新規）
**テスト結果:** 27/27 合格（Sprint 1 の 19 ＋ Sprint 2 の 8）
**バックアップ:** server.py.sprint1.bak（Sprint 1 バックアップ。Sprint 2 は増分）

#### 実装した機能

**画像プレビュー（GET /api/file/raw）**

do_GET の新エンドポイント:

    GET /api/file/raw?session_id=X&path=relative/path

- safe_resolve()（パストラバーサル保護）でワークスペースファイルから生バイトを読む
- 小文字拡張子でキー付けした MIME_MAP 定数から MIME タイプを引く
- 未知タイプは 'application/octet-stream' にフォールバック
- 正しい Content-Type ヘッダで直接バイトを配信
- MAX_FILE_BYTES のサイズ上限なし（画像は大きくなり得る。ブラウザがプログレッシブ読み込みを処理）
- ファイルが無いか、ファイルでない場合 JSON 404 を返す

フロントエンド: openFile() が IMAGE_EXTS セットを確認。画像なら <img src="/api/file/raw?..."> を設定し showPreview('image') を呼ぶ。ブラウザがネイティブに画像を読み込む。onerror ハンドラが読み込み失敗時にステータスメッセージを表示。

**描画済み Markdown プレビュー**

フロントエンドのみ — テキスト内容に既存の GET /api/file エンドポイントを使う。openFile() が MD_EXTS セットを確認。markdown ならテキストを取得し次を呼ぶ:

    $('previewMd').innerHTML = renderMd(data.content);

プレビューは .preview-md コンテナで、チャット吹き出しの .msg-body CSS とは別の完全なタイポグラフィ CSS で描画（狭いサイドパネル向けに異なるサイズ/余白を許す）。

**renderMd() のテーブル対応**

段落ラップ前に正規表現パスを追加:
- row[1] がセパレータ（|---|---|）であるパイプ区切り行のブロックを検出
- <table><thead><tbody> HTML に変換
- 任意の列数を処理
- これは B8（renderMd にテーブルなし）を部分的に解決

**renderFileTree() のスマートファイルアイコン**

新しい fileIcon(name, type) 関数が拡張子を絵文字アイコンにマップ:
- ディレクトリ: フォルダアイコン
- 画像: カメラアイコン
- Markdown: メモ帳アイコン
- Python: ヘビアイコン
- JS/TS/JSX/TSX: 回路アイコン
- JSON/YAML/TOML: 歯車アイコン
- シェルスクリプト: ターミナルアイコン
- その他すべて: ドキュメントアイコン

**タイプバッジ付きプレビューパスバー**

previewPath バーは 2 要素を持つ:
- #previewPathText: 相対ファイルパス
- #previewBadge: タイプラベル付き色付きバッジ（image/md/拡張子）
  画像は青、markdown は金、コードはグレー

#### 追加した新定数

    IMAGE_EXTS   画像拡張子のセット: .png .jpg .jpeg .gif .svg .webp .ico .bmp
    MD_EXTS      markdown 拡張子のセット: .md .markdown .mdown
    CODE_EXTS    参照用のコード/テキスト拡張子のセット
    MIME_MAP     dict: 拡張子 -> MIME タイプ文字列

#### 追加した新 HTML 要素

    #previewPathText   プレビューパスバー内の span（以前は #previewPath の直接 textContent）
    #previewBadge      色付きタイプバッジ span
    #previewImgWrap    プレビュー画像を中央寄せする div
    #previewImg        画像プレビュー用の <img> 要素
    #previewMd         描画済み markdown HTML 用の div

#### エンドポイントリファレンス更新

セクション 18 に追加:

    GET /api/file/raw   ?session_id=X&path=P -> 正しい MIME タイプの生ファイルバイト。
                        パストラバーサル保護。無ければ 404 JSON。

#### B8 状態更新（セクション 9）

B8（renderMd にテーブルなし）は今や PARTIAL: Sprint 2 でテーブルパース追加。入れ子リストと複雑なインライン HTML は依然未処理。完全修正は Phase E（renderMd を marked.js に置換）に残る。


### Sprint 3（2026年3月30日）: パネルナビゲーション ＋ 機能ビューア

**トラック:** バグ修正（3）、機能（3 パネル ＋ 8 API エンドポイント）、アーキ Phase D（部分）
**テスト:** 48/48 合格
**バックアップ:** server.py.sprint2.bak

#### 新しいサイドバーナビゲーション

サイドバー上部に 4 タブ: Chat（デフォルト）、Tasks、Skills、Memory。`.nav-tab` / `.panel-view` CSS クラスで実装。`switchPanel(name)` が正しいタブと panel-view をアクティブにし、初回オープン時にパネルデータを遅延読み込み。

#### Tasks パネル（Cron ビューア）

`loadCrons()` が GET /api/crons を取得、各ジョブを折りたたみ可能な `.cron-item` として描画。`toggleCron(id)` がボディを展開/折りたたみ。`loadCronOutput(jobId)` が各ジョブの GET /api/crons/output から最後の出力ファイルを自動読み込み。

Run Now: POST /api/crons/run がデーモンスレッドでジョブを開始、即座に返す。
Pause/Resume: POST /api/crons/pause と /api/crons/resume が cron.jobs 関数を呼ぶ。

#### Skills パネル

`loadSkills()` が GET /api/skills を取得、`_skillsData` にキャッシュ。`renderSkills()` がカテゴリ別にグループ化、検索入力でフィルタ。skill クリックで `openSkill(name)` が GET /api/skills/content を取得し `showPreview('md')` で右パネルに描画。

#### Memory パネル

`loadMemory()` が GET /api/memory を取得（~/.hermes/memories/ から MEMORY.md ＋ USER.md、~/.hermes/ から SOUL.md を読む）、両方を renderMd() でタイムスタンプ付き markdown として描画。

#### 新 API エンドポイント（セクション 18 更新）

    GET  /api/crons              cron.jobs.list_jobs(include_disabled=True) からの全ジョブ
    GET  /api/crons/output       ?job_id=X&limit=N -> ジョブの最後の N 個の出力 .md ファイル
    POST /api/crons/run          {job_id} -> デーモンスレッドで run_job() をトリガー
    POST /api/crons/pause        {job_id} -> pause_job(job_id)
    POST /api/crons/resume       {job_id} -> resume_job(job_id)
    GET  /api/skills             tools.skills_tool.skills_list() 経由の全 skills
    GET  /api/skills/content     ?name=X -> skill_view(name) 経由の完全な skill データ
    GET  /api/memory             MEMORY.md ＋ USER.md ＋ SOUL.md の内容と mtime

#### 適用した Phase D 入力検証

    require(body, *fields)   フィールド欠如時にクリーンなメッセージで ValueError を投げる
    bad(handler, msg, status=400)  クリーンな JSON エラーレスポンスを返す

堅牢化したエンドポイント: /api/session/update、/api/session/delete、/api/chat/start。
/api/session/update の未知セッション ID は今や 500 ではなく 404 を返す。

#### バグ修正詳細

B6: `newSession()` が `inheritWs = S.session?.workspace` を /api/session/new に渡す。
    バックエンドは session/new で `workspace` パラメータを既に受け付けていたが送られていなかった。

B10: `es.addEventListener('tool', ...)` がステータス更新前に `removeThinking()` を呼び、
     コンパクトな `.msg-role + .msg-body` ツール実行行を表示。`ensureAssistantRow()` も
     最初のトークン到着時に `#toolRunningRow` を除去。

B14: グローバルスコープの `document.addEventListener('keydown', ...)` が Cmd/Ctrl+K を捕捉し、
     busy でなければ `newSession()` を呼ぶ。


### Sprint 4（2026年3月30日）: 移転 ＋ セッションパワー機能 ＋ Phase A/B

**トラック:** バグ（B12、B8、TD5）、機能（リネーム、検索、ファイル操作）、アーキ（Phase A/B 開始）、移転
**テスト:** 68/68 合格
**バックアップ:** server.py.sprint2.bak（最後の完全バックアップ。Sprint 3 と 4 は増分）

#### ソース移転

<agent-dir>/webui-mvp/ を <repo>/ へ移動。
シンボリックリンク: <agent-dir>/webui-mvp -> <repo>
このシンボリックリンクにより既存の全インポートパス（hermes-agent モジュールの sys.path.insert）が変更なしで動作し続ける。start.sh を新しい正典パスを参照するよう更新。

安全: hermes-agent リポジトリの git pull、git reset --hard、git stash から。
非安全: git clean -fd（シンボリックリンクを削除するがターゲットは消さない）。
ディスク障害: 依然シングルコピーのリスク。準備ができたら git init ＋ push を使う。

#### Phase A: CSS 抽出

<repo>/static/style.css: Python 生文字列からの 23KB CSS ブロック。
server.py はもう CSS を含まない。GET /static/* ハンドラがディスクファイルを配信。
server.py は約 200 行縮小。

#### Phase B: セッションごとエージェントロック

SESSION_AGENT_LOCKS = {} は session_id でキー付け、各値は threading.Lock()。
_get_session_agent_lock(sid) がロックを返し、必要なら作成。
_run_agent_streaming() が環境変数ブロックを with _agent_lock: ... でラップ。
これにより同じセッションへの 2 つの並行リクエストが実行途中に環境変数を上書きするのを防ぐ。異なるセッションへの 2 つの並行リクエストは依然非安全（環境変数はプロセスグローバル）。完全修正には環境変数使用の完全除去を要する（Phase B 完了）。

#### 新エンドポイント

    GET  /static/*             <repo>/static/ からファイルを正しい Content-Type で配信。
                               現在 style.css を配信。
    POST /api/session/rename   {session_id, title} -> {session: compact}。80 文字に切り詰め。
    GET  /api/sessions/search  ?q=X -> title が q を含むセッション（大小無視）。
                               空の q は全セッションを返す（/api/sessions と同じ）。
    POST /api/file/delete      {session_id, path} -> {ok: true}。パストラバーサル保護。
    POST /api/file/create      {session_id, path, content?} -> {ok, path}。存在時はエラー。


### Sprint 5（2026年3月30日）: Phase A 完了 ＋ ワークスペース ＋ 編集 ＋ コピー

**トラック:** アーキ（Phase A 完了、TD1/TD2/TD6/Phase C）、機能（3）、テスト（18）
**テスト:** 86/86 合格

#### Phase A 完了: static/app.js

server.py の HTML 文字列から 902 行の JavaScript を <repo>/static/app.js に抽出。
server.py は今や: Python コード ＋ 薄い HTML スケルトン（約875行、1778 から減少）。
レイアウト: server.py は static/ から何もインポートしない。HTML は <link> と <script src> のみ。
Sprint 4 で追加した GET /static/* ハンドラ経由で配信。
node --check が各スプリントで app.js を検証。

#### TD2: LRU SESSIONS キャッシュ

SESSIONS を collections.OrderedDict に変更。
get_session(): ヒット時 SESSIONS.move_to_end(sid)。ミス時: ディスクから読み、追加、move_to_end、SESSIONS_MAX=100 超過でエビクト。
new_session(): 挿入時に同じエビクションロジック。
結果: セッション数によらずメモリ使用が上限化。

#### TD1: スレッドローカル環境コンテキスト

_thread_ctx = threading.local() をサーバーグローバルに追加。
_set_thread_env(**kwargs) と _clear_thread_env() が _thread_ctx.env を設定/クリア。
_run_agent_streaming() が環境変数書き込み前に _set_thread_env() を、外側の finally で _clear_thread_env() を呼ぶ。
プロセスレベルの os.environ 書き込みはフォールバックとして依然存在（terminal ツールがスレッドローカルを読むまで必要）。

#### Phase C: セッションインデックスファイル

SESSION_INDEX_FILE = SESSION_DIR / '_index.json'。
_write_session_index(): SESSIONS ＋ ディスクファイルから compact() リストを構築、JSON を書く。
Session.save() で呼ばれる — インデックスを常に最新に保つ。
all_sessions(): まずインデックス JSON を読む（1 ファイル読み）。インメモリ SESSIONS を重ねる。エラー時は完全 glob スキャンにフォールバック。
再帰を避けるため、'_' で始まるインデックスファイルは完全スキャン中スキップ。

#### 新ワークスペース基盤

WORKSPACES_FILE = ~/.hermes/webui-mvp/workspaces.json
LAST_WORKSPACE_FILE = ~/.hermes/webui-mvp/last_workspace.txt
load_workspaces() / save_workspaces() / get_last_workspace() / set_last_workspace() ヘルパー。
new_session() は今や DEFAULT_WORKSPACE ではなくデフォルトとして get_last_workspace() を呼ぶ。
set_last_workspace() を /api/session/update と /api/chat/start で呼ぶ。

#### 新エンドポイント（Sprint 5）

    GET  /api/workspaces           {workspaces: [...], last: path}
    POST /api/workspaces/add       {path, name?} — 存在＋ディレクトリを検証、重複なし
    POST /api/workspaces/remove    {path} — リストから削除、存在しなくても ok
    POST /api/workspaces/rename    {path, name} — 表示名を更新、無ければ 404
    POST /api/file/save            {session_id, path, content} — 既存ファイルにテキストを書く


### Sprint 6（2026年3月31日）: 磨き込み ＋ リサイズ ＋ Cron 作成 ＋ Phase E

**テスト:** 106/106 合格
**バックアップ:** server.py.sprint5.bak

#### Phase E 完了: static/index.html

HTML ＝ r 三重引用符文字列（197 行、12682 文字）を <repo>/static/index.html に抽出し、リクエストごとにディスク読みで配信。
server.py は今や純 Python: インライン HTML/CSS/JS ゼロ。全静的コンテンツは static/ に。

静的ファイルレイアウト（最終）:
  static/index.html  （Sprint 6）— HTML テンプレート
  static/style.css   （Sprint 4）— 全 CSS
  static/app.js      （Sprint 5）— 全 JavaScript

server.py 行数の推移: 1778（S1）-> 1042（S5）-> 903（S6）

#### Phase D 完了

/api/approval/respond: session_id の存在を検証。choice は (once, session, always, deny) のいずれかでなければならない。不正で 400 を返す。
/api/file/raw: session_id の存在を検証。try/except KeyError で 404 を返す。

#### 新エンドポイント

    POST /api/crons/create   {prompt, schedule, name?, deliver?, skills?, model?}
                             -> {ok: true, job: {...}} または不正なスケジュール/欠如フィールドで 400。
                             cron.jobs.create_job() を直接使う。
    GET  /api/session/export ?session_id=X
                             -> Content-Disposition: attachment ヘッダ付きの完全なセッション JSON。
                             全 messages、workspace、model、タイムスタンプを含む。

#### リサイズ可能パネル

ブート IIFE から _initResizePanels() を呼ぶ。#sidebarResize と #rightpanelResize に mousedown リスナーを作成。mousemove 時: デルタを計算し min/max にクランプ。mouseup 時: 幅を localStorage に保存。幅はブート時に localStorage.getItem() で復元。
CSS: .resize-handle に position:absolute、width:5px、cursor:col-resize。
ドラッグ中はテキスト選択を抑制するため body.resizing を追加。


## ワークスペースパスの信頼レベル

`api/workspace.py` は 2 つの異なる信頼関数を持ちます — 統合しないでください:

**`validate_workspace_to_add(path)`** — `/api/workspaces/add`（明示的なユーザー登録）が使用。
寛容: 非存在、非ディレクトリ、システムルートのパスのみをブロック。ユーザーは意識的に外部パス（例: WSL の `/mnt/d/Projects`）を登録しているので、意図を信頼する。

**`resolve_trusted_workspace(path)`** — 既存ワークスペース内の実際のファイル読み書き操作に使用。
厳格: パスは home 配下、保存済みワークスペースリスト内、または `BOOT_DEFAULT_WORKSPACE` 配下でなければならない。パストラバーサルと不正なファイルアクセスを防ぐ。

この区別が重要なのは、add が循環依存を避けるため寛容な検証を使うからです: 追加するのに保存済みリストが必要なら、保存済みリストにパストを入れられません。

---

> 本ドキュメントは hermes-webui 公式リポジトリのルート `ARCHITECTURE.md` の日本語訳です。コード・関数名・エンドポイント・環境変数・ファイルパス・バージョン番号・PR/バグ ID・識別子は原文のまま表記しています。インデントされたコード/インベントリブロックは、識別子を原文のまま保ちつつ説明文を日本語化しています。情報は原文時点のスナップショットであり、正は公式リポジトリの原文です。
