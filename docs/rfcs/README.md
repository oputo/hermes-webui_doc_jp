# RFC

このディレクトリは、実装の前（または並行）に文章で考え抜く価値のある hermes-webui 機能の設計ドキュメントを保持します — 典型的には、変更が耐久性、復旧、スキーマ、横断的インフラに触れるときです。

## 規約

- RFC ごとに1ファイル。ファイル名は番号ではなくトピック（kebab-case）。
- すべての RFC の冒頭に小さなヘッダを付ける:

      - **Status:** Proposed | Accepted | Implemented | Withdrawn
      - **Author:** @github-handle
      - **Created:** YYYY-MM-DD

- セクションは通常: Problem、Goals、Non-goals、Proposal、Open questions、Rollout plan を含む。該当しないものはスキップ。
- RFC はレビューの出発点。コメントと改訂は別の議論スレッドではなく PR 編集経由で着地する。
- RFC は設計の方向性を文書化する。その断片に対して実装 PR を提出する **招待ではない**。受理された RFC を実装する PR を開く前に、追跡 issue でメンテナに、その実装スライスが望まれているか、他のコントリビューターが既に作っていないかを確認すること。確認された統合先のない RFC 断片の投機的実装は保留される。

## いつ RFC を提出するか

- 変更が、コードを書く前に合意を得たいほど大きい。
- 変更が data-at-rest 形式や復旧セマンティクスに触れる。
- 変更が、他の機能が築く新しいアーキテクチャプリミティブ（ジャーナル、キュー、スケジューラ、キャッシュ層）を導入する。
- レビュアーがコードレビュー中に求める。

迷ったら、ただコードを出荷してください — 小さな機能に RFC は不要です。初回コントリビューターの RFC は、PR を開く前に issue で議論すべきです。

## 現在の RFC

- [`hermes-run-adapter-contract.md`](hermes-run-adapter-contract.md) — #1925 WebUI 実行を明示的なアダプタ境界の背後に移すための、イベント/制御コントラクト、ランタイム状態の所有権マトリクス、受け入れカタログ、可逆的な移行ゲート。
- [`webui-run-state-consistency-contract.md`](webui-run-state-consistency-contract.md) — #2361 アクティブおよび復旧した WebUI run の間、トランスクリプト、モデルコンテキスト、ライブストリーム、リプレイ、圧縮、セッションメタデータを一貫させ続ける一貫性ルール。
- [`live-to-final-assistant-replies.md`](live-to-final-assistant-replies.md) — #3400 長時間のアシスタント応答、ライブプロセスの文章、ツール活動、復旧、終端結果、最終回答の境界のプロダクトモデル。
- [`canonical-session-resolution.md`](canonical-session-resolution.md) — #2361 URL、クエリパラメータ、localStorage、サイドバー、圧縮系統のセッション ID を1つの正典の可視チャットターゲットに解決する焦点を絞ったコントラクト。
- [`turn-journal.md`](turn-journal.md) — 中断されたチャット送信を復旧するためのクラッシュ安全な WebUI ターンジャーナル。

---

> 本ドキュメントは hermes-webui 公式リポジトリ `docs/rfcs/README.md` の日本語訳です。ヘッダテンプレート・ステータス語・Issue 番号・ファイル名は原文のまま表記しています。
