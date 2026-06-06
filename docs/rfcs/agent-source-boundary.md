# エージェントソース境界と API 分離インベントリ

- **Status:** Proposed
- **Created:** 2026-05-17
- **Tracking issue:** [#2453](https://github.com/nesquena/hermes-webui/issues/2453)

## Problem

WebUI は現在、Hermes Agent の Python ソースがランタイムでインポート可能であることに依存しています。ローカルインストールでは通常、隣接するチェックアウトを意味し、マルチコンテナ Docker 構成では、エージェントコンテナも使う `hermes-agent-src` ボリュームを WebUI が読むことを意味します。

そのソースマウントは互換性の橋渡しであり、望ましい長期的なコントラクトではありません。WebUI 側で read-only にマウントされていても、WebUI リリースを Hermes Agent の内部モジュール構成に結合させ、マルチコンテナ構成を実際以上に隔離されているように見せます。

## 現在の安全姿勢

- マルチコンテナ compose ファイルはデフォルトで `hermes-agent-src` を WebUI サービスに read-only でマウントします。
- `docker_init.bash` は、read-only マウントが起動を壊さないよう、`chown` からエージェントソースのサブツリーを剪定します。
- オペレーターが compose ファイルを mutable なエージェントソースマウントで上書きした場合、起動時に注目すべき警告を出します。ローカル開発チェックアウトやカスタムデプロイは意図的に書き込み可能な場合があるため WebUI は依然起動しますが、警告が減じた境界を明示します。

## ソースアクセスインベントリ

これらは、Agent ソースまたは `hermes_cli`/`agent` モジュールがインポート可能であることに依然依存している現在の WebUI 機能です。各項目は最終的に、ライブのソースチェックアウトのマウントを要しない、明示的でバージョン管理された Agent API またはパッケージ化されたライブラリコントラクトの背後に移すべきです。

| WebUI 機能 | 現在の依存 | 望ましい API / コントラクト | 注記 |
|---|---|---|---|
| ブラウザチャット実行 | `api/streaming.py` がインポートする `run_agent.AIAgent` | Run ライフサイクル API: start、observe、status、cancel、approval、clarify、final usage | [#1925](https://github.com/nesquena/hermes-webui/issues/1925) のランタイムアダプタ移行でカバーされるが、今日も依然ソースバック。 |
| ランタイムイベント描画 | Agent の token/reasoning/tool イベント周りの WebUI コールバック | tokens、reasoning、progress、tool ライフサイクル、approvals、clarify、errors、final usage の安定したイベントエンベロープ | 既存の run-adapter RFC がブラウザ向けの形を記述。Agent は依然耐久性のあるプロデューサーコントラクトを必要とする。 |
| プロファイルの list/create/delete/seed | `api/profiles.py` からの `hermes_cli.profiles` | プロファイルメタデータ、env/ランタイムコンテキスト、seed/delete 操作、検証エラーを持つプロファイル管理 API | WebUI は一部操作にフォールバックのファイルシステム処理を持つが、機能同等性は Hermes CLI 内部に従う。 |
| Goal コマンド状態 | `api/goals.py` からの `hermes_cli.goals` | Goal CRUD/制御 API: get、save、pause/resume/clear、status | 直接のモジュールインポートなしで現在の `/goal` WebUI 挙動を保つべき。 |
| スラッシュコマンドレジストリとプラグインコマンド | `api/commands.py` からの `hermes_cli.commands` と `hermes_cli.plugins` | アクティブプロファイルでスコープされたコマンド/プラグイン能力検出 API | WebUI は安定した能力レスポンスからコマンドヘルプを描画すべき。 |
| プロバイダー/認証/モデルカタログ | `api/config.py` からの `hermes_cli.models`、`hermes_cli.auth`、`agent.credential_pool` | プロバイダーレジストリ、モデルカタログ、認証状態、OAuth/credential-pool 状態 API | WebUI は静的フォールバックを持つが、正確な同等性とカスタムプロバイダー状態は Agent 内部から来る。 |
| 伏字化ヘルパーの同等性 | `api/helpers.py` からの `agent.redact.redact_sensitive_text` | 署名/バージョン互換性を持つ伏字化サービス/ライブラリコントラクト | このインポートは過去に変わったことがあるため、WebUI はフォールバックの redactor を保持。 |
| CLI/Gateway セッションブリッジ | サイドバー/セッションヘルパーが読む Agent `state.db` スキーマと gateway メタデータ | 非 WebUI 発のセッションのセッション一覧/トランスクリプト/メタデータ API | 直接の SQLite/スキーマ結合は、特にメッセージング/メール/gateway セッションで、時間とともに狭めるべき。 |

## 分離タスクリスト

1. Docker デフォルトを安全に保つ: WebUI 側の `hermes-agent-src` は 2/3 コンテナ compose ファイルで read-only のままにする。
2. 境界を正直に文書化し続ける: マルチコンテナはプロセス、ネットワーク、リソースを隔離し、ファイルシステム/ソース互換性は隔離しない。
3. Docker で WebUI コンテナが書き込み可能なエージェントソースマウントを見たら大きく警告する。それは多層防御の姿勢を弱めるため。
4. 新しい直接インポートを追加するのではなく、まず #1925 の RuntimeAdapter 経路を通してランタイム実行を変換する。
5. 各インベントリ行について、インポートを置き換える前に Agent API レスポンスの形を定義するフォローアップを起票またはリンクする。
6. チャット実行、プロバイダーカタログ/認証状態、プロファイル、goals、commands/plugins、redaction、インポートされた Agent/Gateway セッションのすべてが安定した置換コントラクトを持つまで、ソースマウントを除去できると主張しない。

## このスライスの Non-goals

- `HERMES_WEBUI_AGENT_DIR` を除去しない。
- ローカルソースチェックアウト開発を壊さない。
- エージェントソースが書き込み可能というだけで起動を失敗させない。
- このドキュメントのみのインベントリスライスで、ランタイムアダプタや Hermes Agent API を置き換えない。

---

> 本ドキュメントは hermes-webui 公式リポジトリ `docs/rfcs/agent-source-boundary.md` の日本語訳です。ステータス・モジュール名・API 用語・Issue 番号・識別子は原文のまま表記しています。
