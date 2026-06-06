# プロジェクトコントラクト

このドキュメントは、既存の Hermes WebUI のコントラクト、RFC、設計制約、レビュー期待に関する、コントリビューター向けの索引です。出典ドキュメントを置き換えるものではなく、提案を実装済みとマークするものでもありません。各リンク先ドキュメントのステータスとスコープに従ってください。

変更を始めるときにこのファイルを使い、コードを編集する前に関連する公開コントラクトを可視化してください。この最初の版はドキュメントルーティングに焦点を当てており、ランタイム挙動、メンテナポリシー、ボット挙動、CI ゲートを変更しません。

## ここから始める

- [`AGENTS.md`](../root-docs/AGENTS.md): AI アシスタント向けリポジトリエントリポイント、公開安全ルール、短いレッドラインチェックリスト。
- [`CONTRIBUTING.md`](../root-docs/CONTRIBUTING.md): コントリビュートスタイル、検証、PR 説明への期待、UI エビデンス、プロジェクト固有の制約。
- [`README.md`](../root-docs/README.md): プロダクト概要、クイックスタート、アーキテクチャマップ、機能インベントリ、docs 索引。
- [`CHANGELOG.md`](../root-docs/CHANGELOG.md): リリースノート用の履歴。メンテナが変更をリリースノートに反映すべきときに更新。（※本翻訳セットでは CHANGELOG は対象外）

## ランタイム・耐久性・状態のコントラクト

- [`docs/rfcs/webui-run-state-consistency-contract.md`](rfcs/webui-run-state-consistency-contract.md): 現行 WebUI のストリーミング、復旧、リプレイ、モデルコンテキスト再構築、圧縮、UI シーン/キャッシュ、サイドバーメタデータ修復について提案される一貫性ルール。既存 WebUI 実行経路を保つ狭い修正はここから始める。
- [`docs/rfcs/live-to-final-assistant-replies.md`](rfcs/live-to-final-assistant-replies.md): 長時間のアシスタント応答、ライブプロセステキスト、ツール活動、復旧、終端結果、最終回答の境界について提案されるプロダクトモデル。実行中セッションのアシスタント応答描画の UI/UX 変更はここから始める。
- [`docs/rfcs/canonical-session-resolution.md`](rfcs/canonical-session-resolution.md): URL ルート、クエリパラメータ、localStorage、サイドバー行、圧縮系統 ID を1つの正典の可視セッションターゲットに解決するための提案コントラクト。セッションルーティング、ブート復元、stale parent、圧縮 tip 選択の変更はここから始める。
- [`docs/rfcs/hermes-run-adapter-contract.md`](rfcs/hermes-run-adapter-contract.md): WebUI 実行をアダプタ境界の背後に移すための、提案されるイベント/制御コントラクト、ランタイム状態の所有権マトリクス、受け入れテストカタログ、可逆的な移行ゲート。アダプタシーム、コントロールプレーン、ランナー、サイドカー、実行所有権の作業に使う。これらのスライスを実装する許可とは扱わないこと。
- [`docs/rfcs/turn-journal.md`](rfcs/turn-journal.md): ブラウザ発のチャットターン向けに提案されるクラッシュ安全な write-ahead ジャーナル。
- [`docs/rfcs/README.md`](rfcs/README.md): RFC 規約と現在の RFC 索引。

変更がストリーミング、復旧、リプレイ、圧縮、コンテキスト再構築、キャンセル、approval/clarify、セッションメタデータ、run 状態に触れる場合、編集前に関連 RFC を読んでください。PR 説明で、影響する状態レイヤーまたはイベント/制御サーフェスを名指しし、該当する不変条件の回帰テストまたは手動検証を含めてください。

提案 RFC はレビューのガードレールであり、実装の許可ではありません。タスクや追跡 issue がそのスライスを明示的に求めない限り、RFC の断片を実装しないでください。

## UI・UX・テーマのコントラクト

- [`DESIGN.md`](../root-docs/DESIGN.md): デザイントークンと現在の calm-console の方向性: 会話優先、静かなメタデータ、控えめなアクセント、デバッグ詳細の段階的開示。
- [`docs/UIUX-GUIDE.md`](UIUX-GUIDE.md): リポジトリの UI/UX 原則の、既存プロジェクト docs とコードコメント由来のコントリビューター向け統合。
- [`docs/ui-ux/index.html`](ui-ux/index.html): 実アプリのスタイルシートに配線されたメッセージエリアインベントリ。
- [`docs/ui-ux/two-stage-proposal.html`](ui-ux/two-stage-proposal.html): issue #536 向けの既存 2 段階チャット UX 提案。
- [`THEMES.md`](../root-docs/THEMES.md): テーマとスキンのガイダンス。中核パレット変数のコントラクトは `static/style.css` にある。

現在の外観はテーマ軸（`light`、`dark`、`system`）と別のスキン軸（`default`、`ares`、`mono`、`slate`、`poseidon`、`sisyphus`、`charizard`、`sienna`、`catppuccin`、`nous`、`geist-contrast`）を `static/boot.js` と `static/style.css` に持ちます。現在のコードとテストがそのモデルがまだ適用されると証明しない限り、古い `data-theme` のみのテーマガイダンスに従わないでください。

UI や UX の作業では、before/after エビデンスを含め、関連するレスポンシブ状態を検証し、一回限りの視覚挙動より安定したクラス/data フックを優先してください。

## 関連コントラクトの選択

編集前に、タスクがどのコントラクトファミリーを行使するか特定してください。これはルーティングチェックであり、リポジトリの全ドキュメントを読む要求ではありません。触れるサブシステムに一致するドキュメントを読んでください。

スコープを明確にするのに役立つとき、issue コメント、draft PR、タスクメモ、AI エージェントのハンドオフで、この軽量なメモを使ってください:

```markdown
## Contract Routing

Task type:
Touched areas:
Relevant public docs:
- `AGENTS.md`
- `CONTRIBUTING.md`
- `docs/CONTRACTS.md`
- <subsystem-specific documents>
Scope boundaries:
Evidence needed before claiming done:
```

小さく明白な修正では、これを短く保ってください。目標はルーティングミスを避けることで、プロセスのオーバーヘッドを作ることではありません。

## コントラクトの変更

コントラクトドキュメント、RFC ガイダンス、コントラクトテストを変更すると、将来のコントリビューターへのレビュー期待が変わります。既存のコントラクトを意図的に変更する PR は、PR 本文に次を含む `Contract Change` セクションを含めるべきです:

- 以前のコントラクト、
- 新しいコントラクト、
- 影響する docs とテスト、
- 互換性または移行の理由。

コントラクトテストと対応する docs は一緒に動かす必要があります。プロダクトセマンティクスをエンコードするテストが、公開 docs を更新せず PR 本文で変更を名指しせずに、反対の挙動をアサートして黙ってコントラクトを再定義してはいけません。

このガイダンスの静的テストはアドバイザリーカバレッジです。ルールが可視のままになるようコントリビューターの文言を固定します。このアドバイザリーカバレッジは自動ポリシーゲートではなく、静的カバレッジは自動ポリシーゲートではなく、GitHub 上で PR 本文の内容を強制しません。将来のリリース時または CI チェックが、PR 本文に `Contract Routing` を欠くコントラクト影響 diff を表面化し得ますが、本ドキュメントはレビュー期待を定義するだけです。

リリースバッチは、含まれるコントラクト影響 PR を明示的に列挙すべきで、レビュアーが通常の green-CI 修正と、プロジェクトのプロダクト/ランタイムガードレールを更新する変更を区別できるようにします。

## PR 準備チェックリスト

PR を開くか更新する前に、`CONTRIBUTING.md` を実際の PR 本文と照合してください。このチェックリストは、コードとテストが既に完了していても適用されます。

必須チェック:

- PR が1つの論理問題を解決する。
- PR 本文が `CONTRIBUTING.md` の全必須セクションを含む: `Thinking Path`、`What Changed`、`Why It Matters`、`Verification`、`Risks / Follow-ups`、`Model Used`。
- `Model Used` がプロバイダー/モデルと注目すべきエージェント/ツール利用を開示、または `None -- human-authored` と記す。
- UI/UX 変更が before/after エビデンスとレスポンシブ状態カバレッジを含む。
- ランタイム/ストリーミング変更が、変更する状態レイヤーまたは不変条件を名指しし、回帰または手動の不変条件チェックを列挙。
- コントラクト影響 PR が `Contract Routing` を含む。意図的なコントラクト変更は `Contract Change` も含む。
- オンボーディング/セットアップ検証が、人間オペレーターが実状態を明示的に要求しない限り、隔離した `HERMES_HOME` と `HERMES_WEBUI_STATE_DIR` を使った。
- docs と `CHANGELOG.md` の更新が含まれるか、明示的に不要とされている。
- GitHub 書き込み後、PR を読み返し、見出しが意図通り描画されたか検証。

green CI と焦点を絞った diff だけでは、PR 説明やエビデンスが触れたサブシステムに一致しなければ不十分です。

## セットアップ・オンボーディング・運用リファレンス

- [`TESTING.md`](../root-docs/TESTING.md): 自動テストコマンドと手動ブラウザテスト計画。
- [`ARCHITECTURE.md`](../root-docs/ARCHITECTURE.md): API、モジュール構成、設計制約。
- [`docs/onboarding.md`](onboarding.md): 初回ウィザードとプロバイダー設定。
- [`docs/onboarding-agent-checklist.md`](onboarding-agent-checklist.md): アシスタント主導のインストール、再インストール、bootstrap、プロバイダー設定、ローカルモデル設定、Docker オンボーディング、WSL オンボーディングの安全ルール。
- [`docs/docker.md`](docker.md): Docker compose セットアップ、よくある失敗、bind-mount 移行。
- [`docs/troubleshooting.md`](troubleshooting.md): よくある失敗の診断フロー。
- [`docs/EXTENSIONS.md`](EXTENSIONS.md): 管理者制御の WebUI 拡張注入。

## クイックレッドラインチェックリスト

変更をレビューに出す前に確認:

- 変更が1つの論理問題を解決する。無関係なリファクタは分離。
- `AGENTS.md`、本索引、触れるサブシステムのリンクされたコントラクトを編集前に読んだ。
- 挙動・セットアップ・アーキテクチャ・テスト・ワークフローの変更が関連 docs を更新。リリースノート用の変更は `CHANGELOG.md` を更新。
- UI/UX 変更が before/after エビデンスを含み、関連するデスクトップ・狭幅・モバイル状態をカバー。
- ランタイム、ストリーミング、復旧、リプレイ、圧縮、サイドバーの変更が、どのレイヤーを変更するか述べ、不変条件の回帰を含む。
- 便益とロールバックの筋書きが明示されない限り、新しい依存・ビルドツール・フレームワーク・長寿命プロセスを避ける。
- オンボーディング/セットアップ検証が、人間オペレーターが実状態を明示的に求めない限り、隔離した `HERMES_HOME` と `HERMES_WEBUI_STATE_DIR` を使う。
- シークレット、プライベートパス、ローカル限定ワークフロー、個人メモがトラッキング対象 docs と例に入らない。

## 将来の進化

この索引は、最初のコントラクトセットを最終にすることを意図していません。将来の PR は、実際の issue、実装変更、RFC 決定、コントリビューターフィードバック、レビュー経験がガイダンスの不完全さや陳腐化を示すとき、コントラクトを追加・改訂・分割・廃止し得ます。

潜在的なフォローアップ領域には、セッションインポート/エクスポート、cron、拡張、セキュリティ境界、Docker/ランタイム隔離、主要なコントラクトリンクのドリフトを防ぐ軽量チェックが含まれます。

---

> 本ドキュメントは hermes-webui 公式リポジトリ `docs/CONTRACTS.md` の日本語訳です。識別子・コントラクト用語・テンプレート（コードブロック内）・環境変数は原文のまま表記しています。ルート文書へのリンクは本翻訳セットの構成（ルート文書を `root-docs/` に配置）に合わせ `../root-docs/...` に調整しています。
