# Hermes Run アダプタコントラクトと移行ゲート

- **Status:** Proposed
- **Author:** @Michaelyklam
- **Updated by:** @franksong2702
- **Created:** 2026-05-11
- **Revised:** 2026-05-23
- **Tracking issue:** [#1925](https://github.com/nesquena/hermes-webui/issues/1925)

## クレジットとスコープ

この RFC は #1925 で議論された方向性を体系化します。実装を導入するものではありません。中心的なガードレールは Michael Lam のレビューの枠組みから来ています:

> アダプタはプロトコル変換器であるべきで、ランタイムの代理ではない。

#1925 からのプロダクト境界は:

> WebUI は実行所有権においては薄くあるべきで、プロダクトスコープにおいて薄くあるべきではない。

つまり WebUI は、セッション、ワークスペースファイル、チャット描画、ツールカード、承認、ステータス、診断、コントロールのための完全なブラウザワークベンチのままです。変わるのは、長寿命の実行所有権が、主 WebUI リクエストプロセス全体に散らばったままでなく、明示的なランタイム境界の背後に移るべきだという点です。

このドキュメントは意図的にレビュー可能な spec かつ移行ゲートです。このアダプタの方向性の実装 PR がストリーミングホットパスを変える、ランナープロセスを導入する、または新しい approval / clarify / queue / goal の制御経路を移す前に、受理されるべきです。新しいランタイム境界を導入しない、現行経路の狭いバグ修正は、WebUI run-state 一貫性コントラクトと該当する issue スコープの下で引き続き進められます。

## Problem

ブラウザ発のチャットターンは依然 WebUI サーバープロセス内で実行されます。現行経路はプロセスローカルなストリーム状態を作り、バックグラウンドエージェントスレッドを起動し、`AIAgent` を構築または再利用し、token、tool、reasoning、approval、clarify、cancellation、terminal イベントのコールバック状態を所有します。

その形は機能しますが、WebUI プロセスをアクティブなランタイム真実の所有者にします。結果:

- WebUI を再起動するとアクティブな作業を孤児化し得る、
- 再接続が、耐久的な run/イベントビューでなくプロセスローカルな状態に依存する、
- 所有権境界の周りでキャンセルや stale writeback のバグが再発する、
- 承認や clarify プロンプトがライブコールバックに結びつく、
- WebUI が単一のアダプタ境界を欠くため、将来の Hermes ランタイム API をきれいに採用できない。

当面の目標はサイドカーを作ることではありません。当面の目標は、ブラウザコントラクトを定義し、現行のランタイム状態を分類し、最初の可逆ジャーナルスライスをゲートすることです。

## 現在のゲート状態 — 2026-05-23

Slice 1 は最初のアクティブ検証ゲートを通過しました:

- #2283 が v0.51.71 で run-journal replay 層を出荷。
- 現行 `origin/master` に対する 100 試行の合成 replay/restart 検証パスが 2026-05-16 に合格。マトリクスは completed-run replay、interrupted stale-pending 復旧、fresh-pending grace 処理、StreamChannel 再接続順序、duplicate-prevention マージ挙動、many-session 復旧、large-journal 導出、stream-to-turn-id ライフサイクルリンクをカバー。
- 焦点を絞った回帰セット `tests/test_turn_journal.py tests/test_turn_journal_lifecycle.py tests/test_stale_stream_pending_recovery.py` も同じ worktree で合格。
- #2393（v0.51.76 で出荷）が、ライブチャットの token SSE トランスポートを選択された会話ペインに制限。バックグラウンドセッションは今やアクティブセッションごとに1つのライブ `/api/chat/stream` EventSource を保つのでなく、既存の status/replay/reattach 挙動に頼る。

この証拠は将来のランナー/サイドカー経路を証明しませんでした。しかしアダプタシーム作業のブロックを解除しました:

- #2416 が Slice 2 の RuntimeAdapter シームコントラクトを出荷。
- #2424 が v0.51.81 でデフォルトオフの `legacy-journal` RuntimeAdapter シームを出荷。
- #2438 が v0.51.83 でレスポンス形状同等性のフォローアップを出荷し、アダプタフラグが `/api/chat/start` の公開 JSON コントラクトを拡張しないようにした。
- #2469 が v0.51.85 で Slice 3a cancel-control ゲートを出荷。
- #2479 が v0.51.86 で最初の Slice 3a 実装を出荷し、`HERMES_WEBUI_RUNTIME_ADAPTER=legacy-journal` が有効なときのみ Stop Generation を `RuntimeAdapter.cancel_run(...)` 経由でルーティング。
- #2487 が Slice 3b の approval/clarify ゲートを出荷し、#2496 が v0.51.89 で approval / clarify レスポンスのアダプタシーム経由ルーティングを出荷。
- #2509 が v0.51.90 で Slice 3c の queue/continue + goal ゲートを出荷。
- #2544 が v0.51.91 で最初の Slice 3c 実装を出荷。goal ルートは今や `HERMES_WEBUI_RUNTIME_ADAPTER=legacy-journal` が有効なときのみ `RuntimeAdapter.update_goal(...)` を使い、legacy-direct なレスポンス形状を保ち、ターン後の goal 評価を既存のエージェントループに残す。
- #2560 が v0.51.92 で queue-staging の明確化を出荷。RFC は今や `queue_message(...)` をステージされたプロトコルメソッドのみとして扱う。`/queue` はブラウザ側の queue/drain 挙動のまま。アダプタ対称性のためだけにサーバー側 queue エンドポイントや queue スケジューラを追加すべきでない。
- #2575 が v0.51.93 で Slice 4a ランナー/サイドカーコントラクトゲートを出荷。
- #2599 が v0.51.94 で Slice 4b の `RunnerRuntimeAdapter` ファサードを出荷。ファサードは、注入されたランナークライアントの start / observe / status / control レスポンスを、`AIAgent`、ストリーム、cancellation フラグ、approval キュー、clarify キュー、goal 状態、cached-agent テーブルを所有せずに正規化する。
- #2627 が v0.51.96 で Slice 4c ランナーバックエンドハーネスゲートを出荷。
- #2696 が v0.51.105 で最初の Slice 4c 実装を出荷: デフォルトオフの `runner-local` アダプタ選択点と、注入されたランナークライアント周りの `build_runtime_adapter(...)` ファクトリ配線。ライブブラウザチャットルートは依然 legacy バックエンドに留まり、監視されたランナープロセスはまだ存在しない。
- #2744 が v0.51.108 で Slice 4d 監視ランナールートゲートを出荷。
- 次の実装スライスは `/api/chat/start` 向けのデフォルトオフのランナールート選択ハーネス。`runner-local` が明示的に選択されたときのみ起動し、監視ランナークライアントが存在するまで bounded な not-configured エラーを返し、`legacy-direct` / `legacy-journal` フォールバックを保ち、WebUI プロセスグローバルを変えるのでなく明示的な profile/workspace/model ペイロードを渡し、新しい名前で `STREAMS` / `CANCEL_FLAGS` / approval キュー / clarify キューを再作成しないべき。

次のゲートはデフォルトでは queue 実装でなくランナーバックエンドの配管です。Queue / continue ルーティングは、将来のメンテナ判断が既存のサーバー側 legacy エントリポイントを特定し、そのレスポンス形状・順序・冪等性コントラクトを固定する場合のみ Slice 4 の前に移すべきです。さもなければ、実行所有権が主 WebUI リクエストプロセスの外へ移る間、`queue_message(...)` をステージのままにするのが誠実な境界です。

## Goals

- 現在の豊かな WebUI ワークベンチ体験を保つ。
- ブラウザ向けのイベント/制御コントラクトを明示する。
- 現行のランタイム所有のすべての状態プリミティブを `runner process`、`journal`、`adapter API surface`、`WebUI presentation cache` のいずれかに分類する。
- 将来のバックエンドマッピングを特定する: 既存の Hermes ランタイム API、欠如している Hermes API、一時的な WebUI 互換シム。
- どの移行も生き延びねばならない受け入れテストを定義する。
- append-only なインプロセスイベントジャーナル / replay 層から始まる、可逆な実装スライスを定義する。

## Non-goals

- この RFC でアダプタを実装しない。
- 最初の実装スライスでランナープロセスやサイドカーを導入しない。
- 最初のジャーナルスライスで `_run_agent_streaming` の制御フローを変えない。
- 新しい名前で `STREAMS`、cached `AIAgent` オブジェクト、コールバックキュー、cancellation フラグを再作成しない。
- WebUI のプロダクトスコープを減らしたり、通常のワークベンチ UX を WebUI の外へ移したりしない。
- WebUI が自身の境界を改善できる前に、Hermes Agent が WebUI 固有のランタイムコネクタを出荷することに依存しない。

## Artifact 1: ブラウザのイベント・制御コントラクト

これは、バックエンドが今日のインプロセスストリーミング経路、インプロセスのジャーナル化経路、将来の WebUI 管理ランナー、将来の Hermes `/v1/runs` バックエンドのいずれであっても、ブラウザが依存する互換性コントラクトです。

現行インベントリは `static/messages.js` のコンシューマと `api/streaming.py` の SSE/イベント生成から導出すべきです。それらのファイルへの将来の編集は、この RFC かそれを置き換える実装コントラクトを更新すべきです。

### イベントエンベロープ

すべての replay 可能なランタイムイベントは次で表現できるべき:

```json
{
  "event_id": "run_123:42",
  "seq": 42,
  "run_id": "run_123",
  "session_id": "20260514_...",
  "type": "tool.updated",
  "created_at": 1778750000.0,
  "terminal": false,
  "payload": {}
}
```

必須セマンティクス:

- `seq` は run ごとに単調。
- `event_id` は SSE `id:` 値または同等のカーソルとして使える程度に安定。
- 再接続は `Last-Event-ID` または `after_seq` をサポート。
- Replay は at-least-once。WebUI は `run_id` + `seq` または `event_id` で重複排除。
- 終端 run はその最終 `done`、`cancelled`、`error` 状態を replay できる。

### イベントファミリー

| イベントファミリー | 必須ペイロード | ブラウザの責任 | ランタイムの source of truth |
|---|---|---|---|
| `run.started` / `status` | ライフサイクル状態、利用可能コントロール、session id、workspace/profile/model/toolset 要約 | アクティブ状態とコントロールを描画 | ランタイム run 状態 |
| `token.delta` | アシスタント message id または segment id、delta テキスト、content type | 可視アシスタントテキストを追記 | ランタイムモデル出力ストリーム |
| `reasoning.delta` / `reasoning.done` | reasoning block id、delta/final テキスト、可視性メタデータ | thinking/progress UI を描画 | ランタイム reasoning イベント |
| `progress` | 簡潔なフェーズ/ステータステキスト、任意のツールコンテキスト | activity/progress テキストを描画 | ランタイム progress コールバック |
| `tool.started` | tool call id、name、サニタイズ済み引数、開始時刻 | ツールカードを開く/更新 | ランタイムツールライフサイクル |
| `tool.updated` | stdout/stderr/構造化部分データ、progress メタデータ | ツールカードを更新 | ランタイムツールライフサイクル |
| `tool.done` | result、status/exit code、duration、error flag | ツールカードを確定 | ランタイムツールライフサイクル |
| `approval.requested` | approval id、アクション要約、リスクメタデータ、利用可能な選択肢 | 承認ウィジェットを表示 | ランタイム approval 状態 |
| `approval.resolved` | approval id、choice、結果ステータス | 承認ウィジェットを閉じる/更新 | ランタイム approval 状態 |
| `clarify.requested` | clarify id、質問、選択肢/入力モード | clarify ウィジェットを表示 | ランタイム clarify 状態 |
| `clarify.resolved` | clarify id、回答メタデータ/ステータス | clarify ウィジェットを閉じる/更新 | ランタイム clarify 状態 |
| `title.updated` | title テキスト、source/confidence | title サーフェスを更新 | session/title サブシステム |
| `usage.updated` / `usage.final` | tokens、cost、model/provider、利用可能なら duration | usage サーフェスを更新 | ランタイム usage 会計 |
| `error` | 安定したエラーコード、安全なメッセージ、伏字化診断、terminal flag | エラーと最終状態を描画 | ランタイム terminal/error 状態 |
| `done` | 最終ライフサイクル状態、usage、terminal result/error 要約、last seq | run UI を確定 | ランタイム terminal 状態 |

### 再接続メタデータ

すべてのアクティブまたは終端 run は次を公開する必要がある:

- `run_id`
- `session_id`
- `status`: `queued`、`running`、`awaiting_approval`、`awaiting_clarify`、`paused`、`cancelling`、`cancelled`、`failed`、`completed`、`expired`
- 最後にコミットされたイベントカーソル / `last_event_id`
- 終了時の terminal 状態と最終 result/error
- 現在利用可能なコントロール
- 保留中の approval/clarify id（あれば）
- 現在の WebUI セッションの session-to-active-run マッピング

### コントロール

| コントロール | 必須セマンティクス | ターゲット所有者 |
|---|---|---|
| observe | ライブイベントにアタッチしカーソルから replay | ランタイム/ジャーナルにバックされた adapter API surface |
| status | SSE/WebSocket が使えないときライフサイクル状態をポーリング | ランタイム/ジャーナルにバックされた adapter API surface |
| cancel | 優雅なキャンセルを要求。terminal イベントが続く | runner/runtime コントロールプレーン |
| queue / continue | Hermes セマンティクスに従ってフォローアップ作業を追記 | runner/runtime コントロールプレーン |
| approval | サポートされる選択肢で id 指定の保留承認を解決 | runner/runtime コントロールプレーン |
| clarify | id 指定の保留 clarify リクエストに回答 | runner/runtime コントロールプレーン |
| goal | 能力が存在する場所で goal を set/status/pause/resume/clear | runtime コマンド/能力プレーン |

WebUI は展開行、選択タブ、ローカルスクロール位置のような表現状態を保ってよい。WebUI はこれらのコントロールのためにランタイム真実を私的に変えてはならない。

## Artifact 2: ランタイム状態インベントリと分類器

分類:

- `runner process`: 主 WebUI リクエストプロセスでなく、最終的な実行ランナー / ランタイムバックエンドが所有すべき。
- `journal`: replay と診断のため append-only な耐久イベントに捕捉すべき。
- `adapter API surface`: 後でバックエンド実装を切り替えられる WebUI 所有の境界を通して公開すべき。
- `WebUI presentation cache`: 実行真実でないためローカルに留まってよい。

| 現行プリミティブ | 現行 legacy の source of truth | ターゲット分類 | 将来のバックエンドマッピング | Slice 1 の扱い | 注記 / ギャップ |
|---|---|---|---|---|---|
| `STREAMS` / `STREAMS_LOCK` | `api.state_sync` プロセスメモリ | adapter API surface + presentation fan-out | WebUI ランナーまたは将来の Hermes run observation API | ライブ経路を保つ。イベントをジャーナルにミラー | アクティブ run の存在の権威であるのをやめる必要。 |
| `CANCEL_FLAGS` | `api.state_sync` プロセスメモリ | runner process | cancel/interrupt エンドポイントまたはランナーコントロール | 制御フロー変更なし | 最終キャンセル状態は replay 可能なイベントとして返る必要。 |
| cached `AIAgent` オブジェクト / `AGENT_INSTANCES` | `api/config.py` プロセスメモリ | runner process | ランナー所有の Hermes 統合 | 変更なし | これの移動はジャーナル証明後まで先送り。 |
| バックグラウンドスレッドライフサイクル | `api/streaming.py` の `_run_agent_streaming` | runner process | ランナー所有の実行ライフサイクル | 変更なし | Slice 1 はスレッド/制御フローを書き直してはならない。 |
| token / 部分テキストバッファ | ストリーミングコールバックとブラウザ SSE 状態 | journal + presentation cache | replay 可能なランタイムイベント | 発行イベントを追記 | ブラウザは描画状態をキャッシュできるが、replay が再構築する必要。 |
| reasoning バッファ | ストリーミングコールバックと UI 描画状態 | journal + presentation cache | replay 可能な reasoning イベント | 発行イベントを追記 | thinking カードは再接続を生き延びる必要。 |
| tool バッファ / ライブツール呼び出し | WebUI ストリーミングコールバック | journal + presentation cache | replay 可能なツールライフサイクルイベント | 発行イベントを追記 | WebUI は描画を所有し、ツール実行状態は所有しない。 |
| approval コールバック / キュー | ライブ Python コールバック | runner process + adapter API surface + journal | approval 状態/制御エンドポイント | request/resolution イベントのみジャーナル | 保留承認は最終的に WebUI 再起動を生き延びる必要。 |
| clarify コールバック / キュー | ライブ Python コールバック | runner process + adapter API surface + journal | clarify 状態/制御エンドポイント | request/resolution イベントのみジャーナル | 保留 clarify は最終的に WebUI 再起動を生き延びる必要。 |
| リクエストごとの `HERMES_HOME` env 変更ロック | `api/streaming.py` / config ヘルパー | runner process | ランナー/プロファイル実行コンテキスト | 変更なし | 長期的にランナーはプロセスグローバル変更なしにプロファイル env を隔離する必要。 |
| session-to-active-run マッピング | session JSON + アクティブ stream id + メモリ | journal + adapter API surface | ランタイム run レジストリ/セッションマッピング | run メタデータをジャーナル | セッション再オープンはアクティブ/完了 run を発見する必要。 |
| title 生成状態 | WebUI コールバック/セッション保存 | journal + presentation cache | ランタイム/セッション title イベント | title イベントを追記 | WebUI はイベント受信後に title 更新を表示してよい。 |
| usage 会計状態 | WebUI コールバック/セッション保存 | journal + presentation cache | ランタイム usage イベント/source of truth | usage イベントを追記 | WebUI のみの乖離した会計を避ける。 |
| コマンド能力メタデータ | WebUI コマンドレジストリ + Hermes コマンド前提 | adapter API surface | ランタイムコマンド/能力メタデータ | 変更なし | 未知のコマンドサポートを WebUI が推測すべきでない。 |
| voice モード状態 | ブラウザ/UI + ストリーミング経路 | presentation cache + adapter API surface | ランタイム入力/制御能力 | 変更なし | 受け入れテストが移行前に voice 挙動を固定する必要。 |
| project/workspace コンテキスト | WebUI session/workspace 状態 + env 変更 | adapter API surface + runner process | ランタイム run コンテキスト | 変更なし | workspace 対応チャットと project コンテキストを保つ必要。 |

未分類の状態は設計のブロッカーです。実装スライスがこの表に合わないランタイムプリミティブを発見したら、コードを着地させる前に RFC を更新してください。

## Artifact 3: 受け入れテストカタログ

これらは移行を生き延びねばならないユーザー観測可能な挙動です。カタログは実用的な範囲で自動テストになるべきです。最初のスライスで完全な自動化が現実的でない場合、PR は最も強力な実用的診断または手動検証計画を含めねばなりません。

| 挙動 | 受け入れ基準 | なぜ重要か | それを証明すべき最初のスライス |
|---|---|---|---|
| 再読み込み/再接続後のジャーナル replay | イベントがジャーナル化された後の再接続や再起動が、重複したトランスクリプト/ツール/reasoning 状態なしにカーソルから replay できる | ブラウザコントラクトが replay 可能で重複安全だと証明 | journal/replay スライス |
| 終端 replay | completed/failed/cancelled run が terminal 状態を replay し、トランスクリプト内容を重複させない | stale スピナーと重複メッセージの回帰を防ぐ | journal/replay スライス |
| 割り込み/stale run 診断 | WebUI が実行を WebUI プロセスが所有中に再起動したら、replay は run が実行し続けたふりをせず、最後にジャーナル化された状態と明確な interrupted/stale 診断を示す | ランナーが存在する前に slice 1 を誠実に保つ | journal/replay スライス |
| 実行が WebUI 再起動を生き延びる | アクティブな実行が主 WebUI プロセスより長く生き、再接続がアクティブ run を発見し、順序付き replay が追いつき、cancel のようなコントロールがまだ動く | 実行所有権が実際にリクエストプロセスの外へ移ったと証明 | runner/sidecar または external-runtime スライス |
| ツール呼び出し中のキャンセル | キャンセルが1つの terminal cancelled 状態を発行し stale writeback がない | 歴史的なストリーム所有権レースを捕捉 | control migration スライス |
| reasoning 中のキャンセル | 部分/reasoning 内容がきれいに保たれ、最終状態が provider-error でない | キャンセル分類の回帰を捕捉 | control migration スライス |
| approval request/response | approval が observation を生き延び、ブラウザ応答がランタイムに届き、結果が replay 可能 | approval コールバックは横断的で孤児化しやすい | approval migration スライス |
| clarify request/response | clarify が observation を生き延び、ブラウザ応答がランタイムに届き、結果が replay 可能 | approval と同じリスク、異なる UI/制御経路 | clarify migration スライス |
| スラッシュコマンド | `/compress`、`/branch`、`/retry` その他サポートコマンドが現在のセマンティクスを保つ | コマンド挙動を ad hoc に再実装すべきでない | command capability スライス |
| セッション途中のモデル切り替え | provider/model の変更が正しいランタイムコンテキストを経由 | provider/source-of-truth のドリフトを防ぐ | adapter control スライス |
| ワークスペースコンテキスト | run がセッション workspace と添付コンテキストを受ける | ワークベンチの価値を保つ | adapter control スライス |
| マルチプロファイル隔離 | プロファイル固有の run が正しい Hermes home とメモリを読み書き | #2134 系の隔離懸念を保護 | runner/profile スライス |
| Queue/continue | ライブ/再開可能な作業中のフォローアップ入力が Hermes セマンティクスに従う | 並行継続モデルを防ぐ | control migration スライス |
| Goal 継続 | goal status/control がアダプタ境界を生き延びる | goal ロジックはライフサイクル敏感 | goal capability スライス |
| Voice モード | voice 発の入力が同じ run/event/control コントラクトを使う | 代替入力経路のドリフトを防ぐ | adapter parity スライス |
| Projects コンテキスト | project メタデータが run replay をまたいで可視で正確なまま | session/ワークベンチの整理を保つ | adapter parity スライス |

## Artifact 4: スライス計画と可逆性

### Slice 0: Spec PR

スコープ:

- この RFC 更新、
- ランタイム挙動の変更なし、
- ストリーミングホットパスのコード変更なし。

revert 経路: docs PR を revert。

### Slice 1: legacy 経路の隣に append-only ジャーナル/replay

この spec が #1925 でレビュー・受理された後にのみ事前承認。

スコープ:

- 既存コールバック経路と並べて append-only イベントジャーナルを追加、
- Artifact 1 のイベントファミリーを捕捉、
- run メタデータ、カーソル、terminal 状態、安全な診断フィールドを永続化、
- 再接続がカーソルから replay し、その後ライブ observation を続けられるようにする、
- `_run_agent_streaming` の制御フローを不変に保つ、
- cancellation、approval、clarify、queue、goal の挙動を不変に保つ。

Non-goals:

- ランナープロセスなし、
- サイドカーなし、
- 制御フローを変えるアダプタインターフェースなし、
- ライブ配信経路としての `STREAMS` の置き換えなし、
- エージェント構築/キャッシュの投機的書き直しなし。

revert 経路:

- 1つの小さな統合シームの背後でジャーナル書き込み/replay を無効化、
- legacy WebUI ストリーミング経路を不変に保持。

成功基準:

1. 非自明な WebUI run を開始。
2. ブラウザを再読み込み/再接続、またはイベントが既にジャーナル化された後に WebUI を再起動。
3. ジャーナルメタデータから run を再発見。
4. 重複した可視トランスクリプト内容なしにカーソルから replay。
5. 再接続なしでワークベンチが描画したであろう、同じジャーナル化済みの token/reasoning/tool/status/terminal 状態を描画。
6. WebUI が実行を WebUI プロセスが所有中に再起動したなら、アクティブ run が実行し続けたと主張するのでなく、明示的な interrupted/stale 診断を示す。

### Slice 2: ジャーナル化された legacy 経路の上のアダプタインターフェース

2026-05-17 時点のステータス: 出荷済み。PR #2416 がアダプタシームコントラクトを定義し、PR #2424 がデフォルトオフの `LegacyJournalRuntimeAdapter` シームを追加し、PR #2438 が `/api/chat/start` のレスポンス形状を `legacy-direct` とフラグ付き `legacy-journal` 経路で同一に保ちました。Slice 2 はサイドカーや実行所有権の移動でなく、可逆な境界変更のままです。

スコープ:

- Slice 1 が replay を証明した後にのみ `RuntimeAdapter` インターフェースを導入、
- 最初のバックエンドを、依然 legacy な経路＋ジャーナルの上の薄いファサードとして実装、
- ブラウザイベントコントラクトを安定に保つ、
- 後の制御固有スライスまでコントロールを既存コードにルーティングし続ける。

revert 経路: フィーチャーフラグを直接 legacy 経路に戻す。

#### Slice 2 インターフェースコントラクト

Slice 2 シームは、実行バックエンドを変えずに意図的に小さな `RuntimeAdapter` 境界を導入すべきです。最初の実装は `LegacyJournalRuntimeAdapter` で、既存の WebUI 所有ストリーミング経路に委譲し、status/replay のため Slice 1 ジャーナルを読みます。これはアダプタを、新しいランタイム所有者でなく、現在のバックエンド上のプロトコル変換器にします。

最小インターフェース形:

```python
class RuntimeAdapter:
    def start_run(self, request: StartRunRequest) -> RunStartResult: ...
    def observe_run(self, run_id: str, *, cursor: str | None = None) -> RunEventStream: ...
    def get_run(self, run_id: str) -> RunStatus: ...
    def cancel_run(self, run_id: str) -> ControlResult: ...
    def respond_approval(self, run_id: str, approval_id: str, choice: str) -> ControlResult: ...
    def respond_clarify(self, run_id: str, clarify_id: str, response: str) -> ControlResult: ...
    def queue_message(self, run_id: str, message: str, *, mode: str = "queue") -> ControlResult: ...
    def update_goal(
        self,
        session_id: str,
        action: Literal["set", "pause", "resume", "clear", "status", "edit"],
        text: str | None = None,
    ) -> ControlResult: ...
```

`queue_message` は legacy のキュー済みメッセージペイロード形にちなんで名付けられています: 任意のランタイム入力でなくフォローアップのチャットテキストを受け付けます。メソッド名は新しい HTTP ルートを要しません。今日 `/queue` は主にブラウザ側の queue/drain 挙動です。アダプタメソッドはプロトコルに入り、後の queue/continue スライスが型付き制御サーフェスを持てるようにしますが、ルート配線は、正確な legacy エントリポイントと順序/冪等性コントラクトが明示されるまで意図的にステージのままです。

`update_goal` では、`action` 引数が bounded なアダプタ能力ラベルです。legacy-journal スライス中、legacy goal パーサが依然完全な `text` ペイロードを受け取り、`set <goal text>` の本体のような詳細について権威のままです。将来のスライスは `action` のみから goal セマンティクスをルーティングしてはなりません。そうすると goal 本体を落とし `/api/goal` 挙動を変えてしまいます。

必須データクラス / ペイロードフィールド:

| 型 | 必須フィールド | 注記 |
|---|---|---|
| `StartRunRequest` | `session_id`, `message`, `attachments`, `workspace`, `profile`, `provider`, `model`, `toolsets`, `source`, `metadata` | 新挙動を導入せず現行 `/api/chat/start` 入力をミラー。 |
| `RunStartResult` | `run_id`, `session_id`, `stream_id`, `status`, `started_at`, `cursor`, `active_controls` | `stream_id` は Slice 2 中 legacy stream id のままでよい。 |
| `RunStatus` | `run_id`, `session_id`, `status`, `last_event_id`, `terminal_state`, `active_controls`, `pending_approval_id`, `pending_clarify_id` | ライブ legacy 状態＋ジャーナル/セッションメタデータにバックされる。 |
| `RunEventStream` | Artifact 1 に一致する順序付きイベント、カーソルから再開可能 | 最初は既存 SSE ＋ ジャーナル replay で実装できる。 |
| `ControlResult` | `accepted`, `status`, `event_id`, `safe_message`, 任意の内部 `payload` | コントロールは Slice 2 で既存ハンドラを呼んでよい。公開 HTTP レスポンスは、後の RFC が拡張しない限りアダプタ専用フィールドを漏らしてはならない。 |

インターフェースは意図的にランナーより狭いです。Slice 2 で `AIAgent`、ツール実行、コールバックキュー、cancellation フラグ、approval コールバック、clarify コールバックを所有しません。それらは個々の移行スライスまで legacy 経路に留まります。

#### Slice 2 フィーチャーフラグと revert コントラクト

Slice 2 は1つの WebUI ローカルな設定/環境フラグでガードすべきです。例えば `HERMES_WEBUI_RUNTIME_ADAPTER=legacy-journal` で、シームが証明されるまでデフォルト `legacy-direct`。フラグはルート/アダプタエントリポイントのみを選択:

```text
legacy-direct   -> 現行 /api/chat/start と /api/chat/stream 経路
legacy-journal  -> 同じ legacy 実行経路＋ジャーナルの上の RuntimeAdapter ファサード
```

revert は運用的に退屈であるべき:

1. フラグを `legacy-direct` に戻す、
2. 必要なら WebUI を再起動、
3. 既存のセッショントランスクリプトとジャーナルファイルは可読のまま、
4. 移行やデータ削除は不要。

シームを導入する PR は、デフォルト経路が `legacy-direct` のままで、アダプタフラグが新エントリポイントを選択する唯一の方法だというソースレベルの回帰を含めるべきです。

#### Slice 2 バックエンドマッピング

| アダプタメソッド | Slice 2 バックエンド | 明示的な non-goal |
|---|---|---|
| `start_run` | 既存の chat-start 準備と legacy `_run_agent_streaming` 経路を呼ぶ | `AIAgent` 構築やスレッド所有権を移さない |
| `observe_run` | 既存のライブ SSE fan-out とジャーナル replay カーソルセマンティクスを組み合わせる | 2つ目のレンダラやイベントプロトコルを作らない |
| `get_run` | セッションメタデータ、ライブストリームの存在、ジャーナル terminal 状態から status を導出 | 耐久 run の存在について `STREAMS` を権威にしない |
| `cancel_run` | 既存のキャンセルハンドラ/制御経路に委譲 | キャンセルセマンティクスをまだ再設計しない |
| `respond_approval` | 既存の approval レスポンス経路に委譲 | approval コールバックを主サーバーに新しいアダプタ所有キューとして永続化しない |
| `respond_clarify` | 既存の clarify レスポンス経路に委譲 | clarify コールバックを主サーバーに新しいアダプタ所有キューとして永続化しない |
| `queue_message` | そのスライスが受理されたとき既存の queue/continue 経路に委譲 | 並行継続バッファや run スケジューラをでっち上げない |
| `update_goal` | そのスライスが受理されたとき既存の goal コマンド/制御経路に委譲 | goal 評価や継続所有権をアダプタに移さない |

主 WebUI プロセス内に新しい長寿命キュー、エージェントキャッシュ、cancellation レジストリ、コールバックレジストリを必要とする実装は Slice 2 のスコープ外で、コード着地前に spec 修正になるべきです。

#### Slice 2 受け入れテスト

シームがデフォルトで有効化される前に、テストは少なくとも次を証明すべき:

- `RuntimeAdapter` インターフェースが存在し、全メソッドが `LegacyJournalRuntimeAdapter` で実装されている;
- アダプタフラグが明示的に有効化されない限りデフォルトルートが legacy direct 経路のまま;
- `start_run` が合成リクエストに対し `/api/chat/start` と同じブラウザ向け `stream_id` / `session_id` 形を返す;
- `observe_run(..., cursor=...)` が既存のジャーナル replay 順序と duplicate-prevention 挙動を保つ;
- `get_run` が現行ライブ状態＋ジャーナル/セッションメタデータを使い、live stream、completed、failed、cancelled、stale / interrupted 状態を区別する;
- `cancel_run` が現行のキャンセル経路に委譲し、依然1つの terminal 結果を発行する;
- approval と clarify メソッドが存在するが、移行スライスまで委譲された legacy コントロールとして文書化されている;
- フラグを無効化するとセッションやジャーナルデータを変えずに古いルート経路に戻る。

これらはアダプタシームテストで、runner-survives-restart テストではありません。execution-survives-WebUI-restart ゲートは Slice 4 に先送りのままです。

### Slice 3: 制御の移行

2026-05-18 時点のステータス: Slice 3a の cancel ルーティングが #2479 経由で v0.51.86 に出荷、Slice 3b の approval/clarify ルーティングが #2496 / #2507 経由で v0.51.89 に出荷、Slice 3c の queue/continue + goal ゲートが #2509 経由で v0.51.90 に出荷。Cancel は、既に1つの明確なブラウザ操作、1つのアクティブ run ターゲット、委譲できる既存 legacy ハンドラを持っていたため、最小のコントロールプレーン移行でした。Approval と clarify はその後、ユーザー仲介のコールバックコントロールについて同じプロトコル変換器の形を証明しました。Queue/continue と goal は、既に保留中のコントロールを解決するだけでなく run ライフサイクルセマンティクスを変え得るため、ランナー前の最後の制御移行です。

スコープ:

- まず cancel を移す、
- 次に approval、
- 次に clarify、
- 次に queue/continue と goal コントロール、
- 各コントロールが独自の受け入れテストとロールバック経路を持つ。

revert 経路: コントロールごとのフィーチャーフラグ、またはルートレベルで legacy コントロールハンドラにフォールバック。

#### Slice 3a: Cancel コントロールゲート

最初の制御移行は、現在の legacy cancel セマンティクスを保ちつつ、Stop Generation を `RuntimeAdapter.cancel_run(...)` シーム経由でルーティングすべきです。新しい cancellation レジストリ、ワーカー所有のシグナルテーブル、サイドカー境界を導入すべきではありません。このスライス中、`cancel_run` は依然既存のキャンセル経路上のプロトコル変換器です。

受け入れ特性:

1. **legacy Stop Generation と同じ可視結果。** キャンセルされたターンは依然1つの terminal cancelled/interrupted 状態を発行し、既存のキャンセルコントラクトに従って既にストリームされた部分アシスタント内容を保つ。
2. **アダプタフラグは挙動保存。** `HERMES_WEBUI_RUNTIME_ADAPTER=legacy-journal` で Stop Generation は `RuntimeAdapter.cancel_run(...)` を使う。デフォルト `legacy-direct` 経路では現行ルートがフォールバックとして残る。
3. **新しいランタイム代理状態なし。** 実装は主 WebUI プロセス内に2つ目の `CANCEL_FLAGS` 風マップ、cached `AIAgent` テーブル、長寿命キュー、ローカルコールバックレジストリを追加してはならない。
4. **ジャーナル/ステータスの一貫性。** キャンセル後、replay とセッション再読み込みがターンを stale/unknown でなく cancelled/interrupted と分類し、terminal 状態が Slice 1 が使うのと同じジャーナル/セッション診断サーフェスを通して可視。
5. **冪等な重複キャンセル。** 同じ run に対するキャンセルの繰り返しは安全であるべき: 1つの terminal 結果が記録され、後の試行は余分な terminal イベントを作ったり stale ストリーム状態を蘇らせたりするのでなく、`not-active` のような bounded な `ControlResult` を返す。

推奨回帰カバレッジ:

- フラグ付き cancel 経路がアダプタシームを呼び、デフォルト経路が legacy ルートのままだと証明するルート/ソーステスト;
- `cancel_run` がちょうど1回委譲し、unsupported/not-active な run に対し bounded な `ControlResult` を返すと証明するアダプタユニットテスト;
- アダプタフラグ下、または同等の合成ハーネスでの既存のキャンセル保存スイート実行（例えば partial-output と cancelled-turn status テスト）;
- キャンセルされた run の terminal 状態が再読み込み後もジャーナル/セッションサーフェスから分類可能だという replay/session-load アサーション。

Slice 3a の Non-goals:

- approval や clarify の移行なし;
- queue/continue や goal の移行なし;
- ランナープロセス、サイドカー、execution-survives-WebUI-restart の主張なし;
- アダプタ専用フィールドのための公開 `/api/chat/start` レスポンス形状の拡張なし。

#### Slice 3b: Approval と clarify のコントロールゲート

次の制御移行は approval と clarify を1つのゲートとしてカバーすべきですが、必ずしも1つの実装コミットではありません。それらは別個のブラウザウィジェットですが、アーキテクチャ上同じ高リスクの形を共有します: エージェントループがライブコールバックで一時停止し、ブラウザがユーザー仲介の決定を提示し、ランタイムがコールバック状態を孤児化せず bounded な応答から再開しなければなりません。

Slice 3b 中、`RuntimeAdapter.respond_approval(...)` と `RuntimeAdapter.respond_clarify(...)` は既存の legacy コールバック経路上のプロトコル変換器のままです。主 WebUI プロセス内に2つ目の approval キュー、clarify キュー、コールバックレジストリ、pending-prompt テーブル、ランナー所有の待機ループを作ってはなりません。

受け入れ特性:

1. **legacy approval / clarify と同じ可視結果。** 既存の承認カード、clarify プロンプト、選択肢、拒否経路、再開エージェント挙動がユーザーにとって不変。アダプタフラグはルート/制御エントリポイントのみを変える。
2. **安定したレスポンスコントラクト。** 既存の approval と clarify の HTTP エンドポイントが現行のブラウザ向けレスポンス形状を保つ。内部ステータス文字列、コールバック id、アクティブコントロールメタデータのようなアダプタ専用フィールドは、後の RFC が明示的にコントラクトを拡張しない限り公開レスポンスに漏れてはならない。
3. **bounded な missing-prompt 挙動。** 存在しない、既に解決済み、stale、期限切れの approval/clarify id への応答は、`not-active` / `expired` / `unsupported` のような bounded な `ControlResult` を返す。リクエストをブロックしたり、コールバックを再作成したり、成功経路を合成したりしてはならない。
4. **replay 可能な request と resolution イベント。** approval/clarify の request と resolution イベントはジャーナル可視のままで、再読み込み/再接続が最後の安全な状態を示せる。Slice 3b は、実行がまだインプロセスの間に保留承認が WebUI プロセス再起動を生き延びるようにする必要はない。その特性はランナー/サイドカーゲートに属する。
5. **新しいランタイム代理状態なし。** 実装はアダプタ固有の名前で新しいプロセスローカルグローバルマップ、長寿命キュー、コールバックレジストリを追加してはならない。既存の legacy コールバック経路がルートを満たすのにより多くの状態を必要とするなら、コード着地前に停止しこの RFC を修正する。
6. **冪等な重複応答。** 同じ approve/deny/clarify 応答の繰り返しは安全: ランタイムは保留リクエストに対し最大1つの応答を受け入れ、1つの resolution イベントを記録し、後の試行は run を2回再開せず bounded な not-active/expired ステータスを返す。

推奨回帰カバレッジ:

- フラグ付き approval と clarify の応答経路がアダプタシームを呼び、デフォルト経路が既存 legacy ハンドラのままだと証明するルート/ソーステスト;
- `respond_approval` と `respond_clarify` がちょうど1回委譲し、accepted/not-active/unsupported な `ControlResult` 値を返し、安全でない内部文字列をブラウザレスポンスに決して公開しないと証明するアダプタユニットテスト;
- request と resolution イベントが再接続後も replay 可能で描画可能だというジャーナル/セッションロードアサーション;
- approval と clarify id の重複応答テスト;
- ブラウザコントラクトのドリフトがないと証明するデフォルト legacy モード下の既存 approval/clarify UI/静的テスト。

Slice 3b の Non-goals:

- queue/continue や goal の移行なし;
- ランナープロセス、サイドカー、execution-survives-WebUI-restart の主張なし;
- 現行 legacy コールバックモデル外での保留 approval/clarify コールバックの永続化なし;
- approval リスク分類、許可選択肢、clarify プロンプト UX の変更なし;
- アダプタ専用フィールドのための公開 chat-start/status レスポンス形状の拡張なし。

#### Slice 3c: Queue/continue と goal のコントロールゲート

次の制御移行は、どのコードもそれらのアクションを `RuntimeAdapter` 経由でルーティングする前に queue/continue と goal を仕様化すべきです。別個の実装 PR として出荷してよいですが、両方とも単に保留プロンプトを解決するのでなく現在のユーザーターン後にエージェントが何をするかに影響するため、1つのゲートを共有すべきです。Queue/continue コントロールはライブまたは再開可能な作業に対しフォローアップ入力を追記/スケジュールします。goal コントロールは持続する複数ターンの目的を set、pause、resume、clear、検査します。WebUI が独立してそれらをバッファまたは評価すると、両方とも偶発的に2つ目の継続モデルを作り得ます。

Slice 3c 中、`RuntimeAdapter.queue_message(...)` と `RuntimeAdapter.update_goal(...)` は既存の legacy queue/goal 経路上のプロトコル変換器のままであるべきです。主 WebUI プロセス内に WebUI 所有の run キュー、goal 評価器、継続スケジューラ、エージェントループ、サイドカー代替を作ってはなりません。

`RuntimeAdapter.update_goal(...)` は goal 状態の変更のみを制御します。ターン後の goal 評価と継続の決定は、後のランナー/サイドカースライスが実行所有権を移すまで既存のエージェント会話ループに残ります。Slice 3c はその評価器を WebUI やアダプタに移してはなりません。

受け入れ特性:

1. **legacy queue/continue と goal と同じ可視結果。** 既存の `/queue` と `/goal` のセマンティクス、ブラウザステータス操作、paused/resumed 状態、ターン後継続挙動がユーザーにとって不変。アダプタフラグはルート/制御エントリポイントのみを変える。
2. **安定したレスポンスコントラクト。** 既存の queue/continue と goal の HTTP またはコマンドレスポンスが現行のブラウザ向け形状を保つ。アダプタ専用の run メタデータ、内部ステータス、能力詳細は、後の RFC が明示的にコントラクトを拡張しない限り公開レスポンスに漏れてはならない。
3. **bounded な unavailable-control 挙動。** 欠如 run、unsupported プロファイル、非アクティブセッション、paused/cleared goal、stale なキュー済み継続へのリクエストは `not-active`、`unsupported`、`conflict` のような bounded な `ControlResult` 状態を返す。phantom run を作ったり、死んだストリームを蘇らせたり、誤ったセッションに対し黙って作業をキューしたりしてはならない。
4. **replay 可能なライフサイクル/ステータス証拠。** queue/continue の送信、goal ステータス変更、結果のターン後継続決定が、legacy 経路が既に同等の状態を発行するジャーナル/セッション診断サーフェスを通して可視のまま。Slice 3c は、実行がまだインプロセスの間にキュー済みフォローアップや goal が WebUI プロセス再起動を生き延びるようにする必要はない。そのより強い特性はランナー/サイドカーゲートに属する。
5. **新しいランタイム代理状態なし。** 実装はアダプタ固有の名前で2つ目のプロセスローカルキュー、goal テーブル、スケジューラ、cached-agent レジストリ、継続ループを追加してはならない。既存の legacy 経路が新しい所有権状態なしにルートをサポートできないなら、コード着地前に停止しこの RFC を修正する。
6. **順序と冪等性が明示的。** 同じ queue/continue リクエストの繰り返しは、legacy 経路が既にその挙動を定義していない限りフォローアップ作業を重複させるべきでない。goal の pause/resume/clear/status 操作は繰り返し安全であるべきで、1つの一貫した状態を報告すべき。

推奨回帰カバレッジ:

- フラグ付き queue/continue と goal 経路がアダプタシームを呼び、デフォルト経路が既存 legacy ハンドラのままだと証明するルート/ソーステスト;
- `queue_message` と `update_goal` がちょうど1回委譲し、accepted/not-active/unsupported/conflict な `ControlResult` 値を返し、安全でない内部文字列をブラウザレスポンスに公開しないと証明するアダプタユニットテスト;
- 繰り返す queue/continue と繰り返す goal pause/resume/clear/status 操作の順序/冪等性テスト;
- legacy 経路が現在状態を発行する箇所で、queue/goal 状態が再接続後も診断可能だというジャーナル/セッションロードアサーション;
- ブラウザコントラクトのドリフトがないと証明するデフォルト legacy モード下の既存 queue/goal UI/静的テスト。

Slice 3c の Non-goals:

- ランナープロセス、サイドカー、execution-survives-WebUI-restart の主張なし;
- 耐久的な WebUI 所有のキューや goal スケジューラなし;
- `AIAgent` 構築、ターン後 goal 評価、エージェント継続ループの legacy 経路外への移行なし;
- `/goal` コマンドセマンティクス、queue 順序セマンティクス、サポート能力メタデータの変更なし;
- アダプタ専用フィールドのための公開 chat-start/status レスポンス形状の拡張なし。

### Slice 4: ランナープロセス / サイドカー境界

Slice 4 は、アクティブな実行所有権を主 WebUI リクエストプロセスの外へ移し得る最初のゲートです。どのランナーコードが着地する前にも、docs/test コントラクト PR として始めるべきです。Slice 1 のジャーナル/replay 層は出荷されアクティブ検証に合格し、Slice 2 のデフォルトオフのアダプタシームは出荷され、Slice 3 の cancel/approval/clarify/goal の制御ルーティングがプロトコル変換器パターンを証明しました。Queue は、メンテナが明示的に別個のランナー前 queue ルートを求めない限りステージのままです。

Slice 4 の実装はアダプタを新しいランタイム代理にしてはなりません。ランナー境界はアクティブな実行、プロセス監視、run ライフサイクル、コールバック状態を所有してよいですが、それらの責務は主 WebUI サーバーに散らばったグローバルとして再作成するのでなく、アダプタ/ランナーコントラクトの背後に集約されねばなりません。

スコープ:

- 長寿命の実行を主 WebUI リクエストプロセスの外へ移す、
- ランナーがアクティブな実行状態を所有、
- 主 WebUI サーバーはアダプタ/ジャーナル経由で observe/replay、
- 将来の Hermes CLI/Python/ローカル API または `/v1/runs` バックエンドをアダプタの背後で評価できる。

revert 経路: ランナーバックエンドを無効化しジャーナル化された legacy バックエンドにフォールバック。

#### Slice 4a: ランナーコントラクトゲート

ランナーコードが着地する前に、次をカバーする狭いコントラクトを定義する:

1. **バックエンド選択とロールバック。** 既存の `legacy-direct` と `legacy-journal` 経路は利用可能のまま。新しいランナーバックエンドはフィーチャーフラグ付き、デフォルトオフで、セッションやジャーナルファイルを削除せずにアダプタモードを `legacy-journal` に戻すことで revert 可能。
2. **プロセス所有権。** 主 WebUI リクエストプロセスでなくランナーが、そのバックエンドに割り当てられた run の `AIAgent` 構築/再利用、アクティブ run 実行、cancellation フラグ、approval/clarify コールバック待機状態、ターン後継続評価を所有する。
3. **耐久 observation。** 主 WebUI サーバーは `RuntimeAdapter.observe_run(...)`、`get_run(...)`、ジャーナルカーソルを通して observe する。ランナーが順序付きイベントと terminal 状態を書き終えるのに WebUI 再起動が必要であってはならない。
4. **Restart/reattach 成功基準。** 長時間 run を開始し、`hermes-webui.service` のみを再起動し、セッションを再読み込みし、アクティブまたは終端のランナー所有 run を再発見し、重複トランスクリプト / ツール / reasoning 状態なしにカーソルから replay/追いつき、run がまだアクティブなら cancel を保つ。
5. **コントロール同等性。** Cancel、approval、clarify、goal status/control、受理された queue/continue 挙動が、安定したブラウザレスポンス形状でアダプタメソッド経由でルーティング。unsupported なコントロールは stale なインプロセス状態に黙ってフォールバックするのでなく bounded な `ControlResult` 状態を返す。
6. **プロファイル/ワークスペース隔離。** ランナー起動が、WebUI サーバーでのプロセスグローバル環境変更に頼るのでなく、明示的な profile、workspace、attachments、model/provider、toolset、source メタデータを受ける。

実装前の推奨コントラクトテスト:

- Slice 4 がフィーチャーフラグ付き・デフォルトオフのままだと証明するソース/RFC テスト;
- ランナー/ジャーナル状態を保ちつつサーバープロセスローカル状態を破棄して WebUI 再起動を模擬し、`get_run` と replay が同じ terminal 状態を復旧すると検証する fake-runner アダプタテスト;
- unsupported なランナーコントロールが bounded な `ControlResult` 値を返し legacy `STREAMS` / `CANCEL_FLAGS` 状態にフォールバックしないと証明する control-parity フィクスチャ;
- ランナーリクエストが主 WebUI プロセスのグローバル `os.environ` を変えずに明示的なコンテキストフィールドを運ぶと証明する profile/workspace ペイロードテスト。

Slice 4a の Non-goals:

- legacy インプロセスバックエンドの除去なし;
- デフォルトオンのランナーモードなし;
- 公開 chat-start/status レスポンス形状の拡張なし;
- アダプタ対称性のためだけの新サーバー側 queue エンドポイントやスケジューラなし;
- WebUI がローカルランナー境界を検証できる前に Hermes Agent が `/v1/runs` を出荷することへの依存なし。

#### Slice 4b: ランナーアダプタクライアントファサード

2026-05-20 時点のステータス: #2599 経由で v0.51.94 に出荷。

Slice 4a コントラクト後の最初のコードスライスは、注入されたランナークライアントに委譲する小さな `RunnerRuntimeAdapter` ファサードであるべきです。これはまだランナープロセスそのものではありません。その仕事は、ルート配線やプロセス監視が着地する前にアダプタ向けの正規化ルールを固定することです:

- `start_run` は明示的な session、profile、workspace、attachments、model/provider、toolset、source、metadata ペイロードを運ぶ `StartRunRequest` を転送;
- `observe_run` と `get_run` はランナーレスポンスを `RunEventStream` と `RunStatus` に正規化し、再作成された WebUI サーバーがプロセスローカル `STREAMS` に頼らず同じランナー所有状態を observe できるようにする;
- コントロールは accepted / not-active / unsupported な結果を bounded な `ControlResult` 値に正規化;
- ファサード自体は `AIAgent`、ワーカースレッド、cancellation レジストリ、approval キュー、clarify キュー、goal スケジューラ、サーバー側キューを所有しない。

実装は、後のスライスが実際のランナークライアント/バックエンドと明示的なルート選択を追加するまでデフォルトオフのままです。

#### Slice 4c: フィーチャーフラグ付きランナーバックエンドと restart/reattach ハーネス

2026-05-21 時点のステータス: #2696 経由で v0.51.105 に出荷。コードはデフォルトオフの `runner-local` アダプタ選択点と、注入されたランナークライアント向けファクトリ配線を追加しつつ、ライブブラウザチャットルートを legacy バックエンドに保ちます。restart/reattach ハーネスは、後のスライスが監視ランナープロセスを導入するまで合成/fake-runner ベースのままです。

ファサードが存在した後、次の狭い実装スライスは、まだ通常のブラウザチャットをそのバックエンドにルーティングせずに、実際のランナークライアント/バックエンド選択点と合成 restart/reattach ハーネスを追加すべきです。

スコープ:

- `HERMES_WEBUI_RUNTIME_ADAPTER=runner-local` のような明示的モードの背後に具体的なランナークライアントファクトリを追加しつつ、`legacy-direct` と `legacy-journal` をデフォルト/revert 経路として保つ;
- `StartRunRequest` が、WebUI プロセスグローバル環境変更に頼らずに明示的な session、profile、workspace、attachments、provider/model、toolset、source、metadata フィールドをランナー境界へ運ぶと検証;
- 再作成された WebUI アダプタが、プロセスローカル状態を破棄した後、ランナー所有 status を再発見しランナー/ジャーナルサーフェスから順序付きイベントを replay できると証明;
- コントロールを `ControlResult` 値で bounded に保ち、unsupported なコントロールは stale な legacy `STREAMS` やコールバックキューにフォールバックするのでなく `unsupported` / `not-active` を返す;
- ランナーバックエンドが合格する restart/reattach ハーネスとルート選択を配線するメンテナ承認を得るまで、ライブ `/api/chat/start` 経路を legacy バックエンドに保つ。

Slice 4c の受け入れテスト:

1. **デフォルトオフ選択。** `legacy-direct` がデフォルトのまま。`runner-local` や後のランナーモードは明示的なフィーチャーフラグでのみ選択。
2. **ルート形状ドリフトなし。** ランナーバックエンドの追加が、ルートが legacy バックのまま公開 `/api/chat/start`、cancel、approval、clarify、goal、status のレスポンス形状を拡張しない。
3. **Restart/reattach ハーネス。** fake またはローカルランナーフィクスチャが run を開始し、最初の WebUI アダプタインスタンスを破棄し、アダプタを再作成し、それでも耐久ランナー所有状態から順序付きイベント＋terminal/live status を observe できる。
4. **コントロール境界。** Cancel / approval / clarify / queue / goal コントロールは、ランナーバックエンドが選択されたときのみランナークライアント経由でルーティングし、unsupported なコントロールは legacy プロセスローカル状態を参照せず bounded な `ControlResult` 値を返す。
5. **ランタイム代理グローバルなし。** 主 WebUI サーバーは、ランナー所有ストリーム、cancellation フラグ、保留 approval/clarify コールバック、cached agent、goal/queue スケジューラの新しいモジュールレベルマップを得てはならない。

Slice 4c の Non-goals:

- デフォルトオンのランナーバックエンドなし;
- legacy インプロセスバックエンドの除去なし;
- 公開レスポンス形状の拡張なし;
- restart/reattach ハーネスがレビューされる前のランナーバックエンドへのライブチャットルート切り替えなし;
- アダプタ対称性のためだけのサーバー側 queue エンドポイントやスケジューラなし。

#### Slice 4d: 監視ランナーバックエンドのルートゲート

2026-05-23 時点のステータス: #2744 経由で v0.51.108 に出荷。このゲートは docs/test コントラクトのまま: デフォルトオフのルート選択要件を定義するが、それ自体はライブチャットをランナーバックエンドにルーティングしない。

`runner-local` 選択が存在した後、次のレビュー可能なゲートは、ライブブラウザチャットが使えるようになる前に最初の監視/ローカルランナーバックエンドとルート選択ハーネスを定義すべきです。これはまずコントラクト/test スライスのまま: デフォルトオンのランナーモードなし、`legacy-direct` や `legacy-journal` の除去なし、ハーネスが実証するまで本番 WebUI ターンが再起動を生き延びるという主張なし。

スコープ:

- ランナープロセス/クライアントのライフサイクルを定義: 主 WebUI リクエストプロセスに新しいアクティブ run マップを置かずに、run がどう spawn・監視・observe・終了されるか;
- デフォルトが legacy バックのまま公開レスポンス形状を変えずに `/api/chat/start` のルート選択点を定義;
- ランナー所有イベントが、安定したカーソル、terminal 状態、replay 順序を持つ WebUI ジャーナルイベントになる方法を指定;
- cancellation、approval、clarify、goal、ステージされた queue 挙動をランナークライアントの `ControlResult` レスポンスとして指定し、unsupported なコントロールは legacy プロセスローカルコールバックに黙ってフォールバックするのでなく bounded で可視に;
- ランタイム API ギャップマトリクスを引き継ぐ: active-run 発見、session-to-run ルックアップ、コマンド能力メタデータ、artifact イベント、provider/tool ルーティングのような欠如する Hermes 所有能力は、プライベートな WebUI ランタイムレプリカでなく明示的なギャップまたは一時的なアダプタ状態のままであるべき。

Slice 4d の受け入れテスト:

1. **ルートはデフォルトオフのまま。** `HERMES_WEBUI_RUNTIME_ADAPTER` 未設定と `legacy-direct` が `/api/chat/start` を既存経路に保つ。`runner-local` がランナールートを選択できる唯一のモード。
2. **Restart/reattach ハーネスが所有権の移動を証明。** fake またはローカルランナーが run を開始し、最初の WebUI サーバー/アダプタインスタンスが破棄され、新しいアダプタインスタンスが同じアクティブまたは終端 run を発見し、replay がカーソルから追いつき、run がまだアクティブなら cancel が利用可能のまま。
3. **公開レスポンス形状ドリフトなし。** Chat start とコントロールレスポンスが安定のままで、アダプタ専用フィールドは内部のまま、または新しいコントラクト改訂として明示的に文書化される。
4. **ランタイム代理グローバルなし。** 主 WebUI サーバーは、ランナー所有ストリーム、cancel フラグ、approval/clarify コールバック、cached agent、goal 状態、queue スケジューラの新しいモジュールレベルマップを得ない。
5. **明示的なコンテキストペイロード。** ランナー起動が、WebUI サーバーでのプロセスグローバル環境変更に依存するのでなく、session、profile、workspace、attachments、provider/model、toolsets、source、metadata をペイロードフィールドとして運ぶ。

Slice 4d の Non-goals:

- デフォルトオンのランナーバックエンドなし;
- legacy インプロセスバックエンドの除去なし;
- アダプタ対称性のためだけのサーバー側 queue エンドポイントやスケジューラなし;
- 欠如能力がランナーまたは将来の Hermes Runtime API に属するときの、恒久的な WebUI 所有 active-run 発見キャッシュなし;
- 広範な UI/プロダクトサーフェスの移行なし。WebUI はリッチなワークベンチのままで、実行所有権のみが移る。

#### Slice 4e: デフォルトオフのランナー chat-start ルート選択ハーネス

2026-05-24 時点のステータス: #2794 経由で v0.51.129 に出荷。ルート選択ハーネスは今やアダプタモード選択を明示的にします: `legacy-direct` がデフォルトのまま、`legacy-journal` は依然既存のジャーナル化 legacy 経路に委譲し、`runner-local` はインプロセス legacy run を黙って開始するのでなく bounded な not-configured レスポンスを返します。

Slice 4d ゲート後の最初の実装は、まだ監視ランナープロセスを追加せずに、`/api/chat/start` 選択点を既存の `RuntimeAdapter` ファクトリに配線すべきです。ハーネスは選択挙動を明示的にしなければなりません: `legacy-direct` がデフォルトのまま、`legacy-journal` が legacy インプロセスストリーム経路に委譲し続け、`runner-local` がランナークライアント未設定時に legacy に黙ってフォールバックしない。

スコープ:

- アダプタモードが明示的に選択されたとき `/api/chat/start` を `build_runtime_adapter(...)` 経由でルーティング;
- 成功したブラウザレスポンスを `stream_id`、`session_id`、`pending_started_at`、`turn_id`、`title`、有効な model/provider メタデータのような legacy 互換フィールドにホワイトリスト化;
- 監視ランナークライアント/バックエンドが着地するまで `runner-local` に bounded な not-configured エラーを返す;
- 既存の明示的な `StartRunRequest` ペイロードフィールドをシーム越しに渡す。

Slice 4e の受け入れテスト:

1. **デフォルトは legacy-direct のまま。** アダプタ env var なしで `/api/chat/start` は `_start_chat_stream_for_session(...)` を直接使い続ける。
2. **Legacy-journal は挙動保存のまま。** フラグ付き legacy アダプタは依然同じ stream-start ヘルパーに委譲し公開レスポンス形状を保つ。
3. **Runner-local は黙ってフォールバックしない。** `runner-local` が選択されたがランナークライアントが存在しないなら、ルートはオペレーターの知らぬ間に WebUI 所有の legacy run を開始するのでなく bounded なエラーを返す。
4. **アダプタ内部レスポンスドリフトなし。** `run_id`、`status`、`active_controls` は、後のコントラクトが明示的に公開するまで内部のまま。
5. **ランタイム代理グローバルなし。** ハーネスは主 WebUI プロセスにランナー所有ストリーム、cancel、approval、clarify、cached-agent、goal、queue マップを追加しない。

Slice 4e の Non-goals:

- まだ監視ランナープロセスなし;
- デフォルトオンのランナーモードなし;
- 本番チャットターンの execution-survives-WebUI-restart 主張なし;
- `legacy-direct` や `legacy-journal` の除去なし;
- アダプタ対称性のためだけのサーバー側 queue エンドポイントやスケジューラなし。

#### Slice 4f: 監視ローカルランナークライアントバックエンドゲート

2026-05-31 時点のステータス: #3073 / #3274 経由で v0.51.188 に出荷。クライアントトランスポートは今や `HERMES_WEBUI_RUNNER_BASE_URL` の背後で実装され、デフォルトオフのまま。エンドポイント未設定なら、`runner-local` は依然 bounded な not-configured 経路を返し、ライブのインプロセス `_run_agent_streaming` 経路は不変。設定時、WebUI は start / observe / status / controls に JSON HTTP クライアント境界を使い、主プロセスにランナー所有マップを追加するのでなく、observe したランナーイベントを既存の SSE ストリームルート経由でブリッジします。

このリリースは #3073 を吸収しつつ2つのセキュリティハードニングチェックを追加しました: `HttpRunnerClient` は非 `http(s)` の base URL スキームを拒否し、リダイレクトに従わない opener を使うため、誤設定または侵害されたランナーが Bearer トークンをリダイレクト先ホストに漏らせません。リリースゲートは pytest 完全合格とデフォルトオフ/不活性経路の独立レビューを報告しました。

このブリッジは意図的に WebUI コンシューマトランスポートシームです: 設定されたランナーは、既にブラウザ SSE のイベント名/ペイロードと互換なイベントを発行しなければならないか、後のランナー所有正規化層が `token.delta`、`tool.started`、`done` のような Hermes ランタイムファミリーをこのルートに届く前に変換しなければなりません。

設定されたランナークライアント境界が出荷された後、次のレビュー可能なステップは `runner-local` をデフォルトにすることではありません。そのクライアント境界の背後で実際に `AIAgent` 実行を所有でき、設定された外部エンドポイントや fake-runner フィクスチャだけでなく実ローカルランナーで restart/reattach を証明できる、最初の監視ランナープロセスハーネスを定義することです。

このスライスはクライアント境界の実装ゲートでした。目標は、実装が `api/routes.py` 内で名前を変えただけの `STREAMS` / `CANCEL_FLAGS` / cached `AIAgent` の代理にならないよう、最小のランナークライアント挙動を固定することでした。

スコープ:

- ランナークライアントのプロセス境界とライフサイクルを定義: `start_run` がどう作業を spawn/ハンドオフするか、子がどう監視されるか、主プロセスのアクティブ run 辞書なしに terminal 状態がどう記録されるか;
- 新たに再起動した WebUI プロセスが古い `STREAMS` エントリを参照せずに発見できる、耐久ランナー所有 run id ＋ session-to-run ルックアップを要求;
- 既存のジャーナル/カーソルサーフェス経由の順序付きイベント replay を要求し、token、reasoning、progress、tool、usage、error、done イベントが legacy replay と同じブラウザ経路で描画されるようにする;
- cancel をアクティブなランナー所有 run の最初の必須ライブコントロールとして定義し、approval、clarify、goal、queue を明示的なランナー能力にマップするか bounded な unsupported/conflict な `ControlResult` 値として返す;
- profile、workspace、attachments、provider/model、toolset、source、metadata を、プロセスグローバル WebUI 環境変更に依存するのでなく、ランナー境界での明示的なペイロードフィールドとして保つ。

Slice 4f の受け入れテスト:

1. **501 経路は設定時のみ置き換え。** アダプタモード未設定と `legacy-journal` 挙動は不変のまま。`runner-local` はバックエンドが明示的に設定されたときのみ監視ランナークライアントを使う。
2. **Restart/reattach が所有権の移動を証明。** ランナー所有 run を開始し、WebUI サーバープロセスを破棄/再起動し、耐久ランナー/ジャーナル状態からアクティブまたは終端 run を再発見し、重複なしにカーソルから replay し、run がまだアクティブなら cancel を保つ。
3. **ランタイム代理グローバルなし。** 主 WebUI サーバーは、ランナー所有ストリーム、cancel フラグ、approval/clarify コールバック、cached agent、goal 状態、queue スケジューラ、子プロセス run レジストリの新しいモジュールレベルマップを得ない。監視状態はランナークライアント/バックエンド境界に属する。
4. **安定したブラウザコントラクト。** 成功した chat-start レスポンスは、後のコントラクト改訂が明示的に `run_id`、`status`、`active_controls` を公開しない限り legacy 互換フィールドホワイトリストに制限されたまま。
5. **bounded なコントロールギャップ。** unsupported なランナーコントロールは安全な `unsupported`、`not-active`、`conflict` 結果を返す。ランナー所有 run について legacy コールバックキューにフォールバックしてはならない。

Slice 4f の Non-goals:

- デフォルトオンのランナーモードなし;
- legacy インプロセスバックエンドの除去なし;
- 広範な WebUI プロダクトサーフェスの移行なし;
- アダプタ対称性のためだけのサーバー側 queue スケジューラなし;
- ランナーまたは将来の Hermes Runtime API の責務を複製する恒久的な WebUI 所有 active-run 発見キャッシュなし。

#### Slice 4g: 監視ローカルランナープロセスハーネスゲート

#3073 / #3274 の後、WebUI は明示的な設定済みランナー HTTP クライアントと SSE コンシューマブリッジを持ちますが、監視ランナープロセスそのものはまだ出荷していません。次のゲートは、既に出荷済みの `runner-local` クライアント境界を通して消費されつつ、主 WebUI リクエストプロセスの外で `AIAgent` 実行を所有できる最小のローカルランナーハーネスを定義すべきです。

スコープ:

- ローカルランナープロセスのライフサイクルを定義: spawn/start、ヘルスチェック、run 所有権、優雅なシャットダウン、クラッシュ分類、クリーンアップ;
- WebUI を `HERMES_WEBUI_RUNNER_BASE_URL` のクライアントとして保ち、プロセスローカルなランナー実行状態の所有者にしない;
- run/session ルックアップ、順序付きイベント、terminal 状態、アクティブコントロールを、再起動した WebUI が発見できるランナー所有またはジャーナルバックの状態に永続化;
- 明示的な profile、workspace、attachments、provider/model、toolset、source、metadata ペイロードを WebUI プロセスグローバル環境変更なしにランナーへ運ぶ;
- cancel をアクティブなランナー所有 run の最初のライブコントロールとして証明し、approval、clarify、goal、queue を明示的なランナー能力にマップするか bounded な unsupported/conflict な `ControlResult` 値として返す。

Slice 4g の受け入れテスト:

1. **プロセス所有権が移動。** `hermes-webui` でなくローカルランナープロセスが、`runner-local` run の `AIAgent` 構築/再利用とアクティブ run 実行を所有。
2. **実ランナーでの restart/reattach。** 非自明な `runner-local` run を開始し、`hermes-webui` のみを再起動し、セッションを再読み込みし、アクティブまたは終端のランナー所有 run を再発見し、重複トランスクリプト/ツール/reasoning 状態なしにカーソルから replay/追いつき、まだアクティブなら cancel を保つ。
3. **WebUI にランタイム代理グローバルなし。** 主 WebUI サーバーは依然、ランナー所有ストリーム、cancel フラグ、approval/clarify コールバック、cached agent、子プロセス run レジストリ、goal 状態、queue スケジューラの新しいモジュールレベルマップを得ない。
4. **デフォルトオフで可逆。** `HERMES_WEBUI_RUNNER_BASE_URL` を未設定にするかアダプタモードを legacy に戻すと、セッションやジャーナルの移行なしに既存のインプロセス経路が利用可能のまま。
5. **ランナーのヘルスと失敗が observable。** 欠如・不健全・クラッシュしたランナーは、ランナー選択された run について WebUI 所有実行に黙ってフォールバックするのでなく、bounded な診断と terminal/interrupted 状態を返す。

Slice 4g の Non-goals:

- デフォルトオンのランナーモードなし;
- `legacy-direct` や `legacy-journal` の除去なし;
- アダプタ対称性のためだけのサーバー側 queue スケジューラなし;
- 広範な WebUI プロダクトサーフェスの移行なし;
- これが正典の Hermes Agent Runtime API だという主張なし。Hermes Agent が後で `/v1/runs` を出荷するなら、このローカルランナーは同じアダプタ/クライアント境界の背後で置き換え可能なバックエンドのままである。

## 最初の意味ある成功基準

最初の意味あるマイルストーンは意図的に分けられています。

### Journal / Replay ゲート

このゲートは Slice 1 に属します。このスライスでは実行がまだ WebUI プロセスに所有されているため、アクティブな実行が WebUI プロセス再起動を生き延びることは証明しません。

証明するもの:

1. WebUI run が安定したカーソル付きの append-only ジャーナルイベントを発行する。
2. ブラウザ再読み込み/再接続が、既にジャーナル化されたイベントをカーソルから replay できる。
3. terminal な `done`、`error`、`cancelled` 状態が重複トランスクリプト内容なしに replay する。
4. tool/reasoning/status 状態が replay されたジャーナルイベントから再構築できる。
5. 実行所有権がプロセス外へ移る前に WebUI が再起動すると、UI が最後にジャーナル化された run 状態について明確な interrupted/stale 診断を示せる。

### Execution-Survives-WebUI-Restart ゲート

このより強いゲートは Slice 1 でなくランナー/サイドカーまたは external-runtime スライスに属します。実行所有権が実際に主 WebUI リクエストプロセスの外へ移ったことを証明します:

1. WebUI から長時間 run を開始。
2. `hermes-webui` のみを再起動。
3. 再起動した WebUI プロセスの外でアクティブ run の実行を続ける。
4. ブラウザ/セッションを再読み込み。
5. アクティブ run を再発見しカーソルから replay/追いつく。
6. 重複トランスクリプト内容なしに描画されたワークベンチ状態を保つ。
7. run がまだアクティブなら、cancellation がまだ動く。

これがランタイム所有権を新しいプロセスローカルグローバルの山に移さずに動くなら、アーキテクチャは正しい方向に進んでいます。

## オープンな問い

- Slice 1 はどの正確なストレージ形式を使うべきか: SQLite run/event テーブル、JSONL、トランスクリプト由来チェックポイントのハイブリッド?
- terminal 状態後、イベント replay はどれくらい保持すべきか?
- ジャーナル永続化の前にどのイベントフィールドを伏字化すべきか?
- ジャーナルは WebUI 状態ディレクトリ、セッションディレクトリ、将来のランタイム固有サブディレクトリのどこに置くべきか?
- legacy 描画と replay 描画を比較するのに必要な合成イベントフィクスチャの最小セットは何か?
- どのコントロールが移行前にルートレベルのフィーチャーフラグを必要とするか?
- Hermes Agent が後で耐久 `/v1/runs` API を出荷するなら、どのアダプタフィールドが直接マップし、どれが WebUI 表現の懸念のまま残るか?

---

> 本ドキュメントは hermes-webui 公式リポジトリ `docs/rfcs/hermes-run-adapter-contract.md` の日本語訳です。ステータス・スライス名・メソッド名・データクラス/フィールド名・コード・環境変数・Issue/PR 番号・バージョン番号・識別子は原文のまま表記しています。
