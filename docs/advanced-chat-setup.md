# 高度なチャット設定

セルフホストの Hermes WebUI デプロイ向けの、2つの任意機能です。**ほとんどのユーザーはどちらも不要** です。デフォルト（インプロセスチャット、prefill なし）がそのまま動きます。

## セッションリコール prefill

WebUI は、ブラウザ発の新しいエージェントターンに、一時的な prefill メッセージを付加できます。これは、Joplin、Obsidian、Notion、llm-wiki などのサードパーティのメモソース向けにローカルなリコールやルータースクリプトを既に持つデプロイが、ブラウザチャットに「永続的なコンテキストの所在」を知らせたい場合に有用です。

すべての新しいブラウザセッションにメモコーパス全体を流し込むのではなく、コンパクトなルーター型 prefill（例: 「Joplin に永続的なプロジェクトコンテキストがある。詳細依存の質問に答える前に利用可能な notes/search ツールを使え」）を推奨します。prefill はエージェントを検索（retrieval）へ向けるべきで、具体的な事実はオンデマンドで notes/search ツールが提供すべきです。

静的 JSON は `prefill_messages_file` または `HERMES_PREFILL_MESSAGES_FILE` で引き続きサポートされます。動的リコールには、WebUI 固有のスクリプトフックで明示的にオプトインします:

```yaml
webui_prefill_messages_script:
  - python3
  - /path/to/notes_recall.py
webui_prefill_messages_script_timeout: 5
```

または:

```bash
HERMES_WEBUI_PREFILL_MESSAGES_SCRIPT="python3 /path/to/notes_recall.py" \
HERMES_WEBUI_PREFILL_MESSAGES_SCRIPT_TIMEOUT=5 \
./ctl.sh restart
```

スクリプトは、OpenAI スタイルの JSON メッセージリスト、`messages` リストを持つ JSON オブジェクト、またはプレーンテキストのいずれかを出力できます。プレーンテキストは1つの `user` prefill メッセージとしてラップされ、動的リコールテキストは追加のシステム命令ではなく通常のコンテキストになります。フックがシステムレベルのガイダンスを提供する必要がある場合は、代わりに明示的な `role: "system"` エントリを持つ JSON メッセージを出力してください。スクリプト出力はパース前に 256 KiB で上限が設けられます。パース済みの prefill コンテキストは、その後 `webui_prefill_context_max_chars` または `HERMES_WEBUI_PREFILL_CONTEXT_MAX_CHARS`（デフォルト: 12,000 文字。`0` で無効化）で制限されます。動的スクリプトが予算を超え、コンパクトな静的 prefill ファイルが設定されている場合、WebUI はそのファイルにフォールバックします。コンパクトなフォールバックが無い場合、WebUI は、巨大なメモ/本文ペイロードを新しいブラウザターンのたびに送る代わりに、短い検索指示を注入します。ブラウザはコンパクトなステータスイベント（`source`、`label`、メッセージ数、圧縮メタデータ、伏字化されたエラー）のみを受け取り、prefill メッセージ本文は決して受け取りません。

## Gateway 経由のブラウザチャット

デフォルトでは、ブラウザチャットは WebUI のインプロセスなレガシーランタイムを通ります。高度なセルフホストデプロイは、既存の WebUI `/api/chat/start` と `/api/chat/stream` のブラウザコントラクトを保ったまま、新しいブラウザターンを稼働中の Hermes Gateway API サーバー経由でルーティングするようオプトインできます:

```bash
HERMES_WEBUI_CHAT_BACKEND=gateway \
HERMES_WEBUI_GATEWAY_BASE_URL=http://127.0.0.1:8642 \
HERMES_WEBUI_GATEWAY_API_KEY=... \
./ctl.sh restart
```

`HERMES_WEBUI_CHAT_BACKEND` は意図的に厳格です。`gateway`、`api_server`、`api-server` のみがブリッジを有効化します。`1` や `true` のような一般的な truthy 値は無視され、既存デプロイが誤って実行の所有権を変えないようにします。`HERMES_WEBUI_GATEWAY_API_KEY` が省略された場合、WebUI は存在すれば `API_SERVER_KEY` にフォールバックします。Gateway が HTTP 401 を返すと、WebUI は Gateway の一般的なプロバイダー風 "Invalid API key" 本文を表示する代わりに、この WebUI↔Gateway のキー不一致を指す `gateway_auth_error` を報告します。`/api/health/agent` も伏字化された `gateway_chat` ブロックを含み、オペレーターはキー値を露出せずに gateway モード・base URL・API キーの有無が設定されているか確認できます。その `gateway_chat` フィールドはオペレーター診断用ペイロードに過ぎず、現状ブラウザ UI のユーザー向けヘルスバナーとしては描画されません。

このブリッジは、既に Hermes Gateway/API Server をローカルで動かしていて、ブラウザ発のチャットをメッセージングサーフェスと同じランタイム/ツール経路で使いたいオペレーターに最適です。添付、キャンセル、承認、clarify プロンプトは依然 WebUI の現行の互換性経路に従い、ランタイムアダプタ移行が完了するまではすべてのメッセージングサーフェスと一致しない場合があります。

---

> 本ドキュメントは hermes-webui 公式リポジトリ `docs/advanced-chat-setup.md` の日本語訳です。設定キー・環境変数・APIエンドポイントは原文のまま表記しています。
