# Hermes WebUI 日本語マニュアル（非公式）

このフォルダは、[hermes-webui](https://github.com/nesquena/hermes-webui) 公式リポジトリの `docs`（および一部のルート文書）を日本語訳したものです。原文のリポジトリ構成をミラーし、Markdown を忠実に翻訳しています。

- 翻訳方針: 技術用語・コマンド・パス・環境変数・設定キー・バージョン番号・識別子は **原文のまま英語表記**。説明文を日本語化。
- 注意: 記載のバージョン・日付・挙動は **原文時点の情報** であり、将来変わり得ます。正は常に公式リポジトリの原文です。
- 各ファイル末尾に、対応する原文ファイルへの出典注記を付けています。

> このフォルダは公式の翻訳物ではなく、個人利用のための非公式訳です。

---

## ドキュメント一覧

### 利用者向けマニュアル（`docs/`）

| ファイル | 内容 | 状態 |
|---|---|---|
| [docs/why-hermes.md](docs/why-hermes.md) | Hermes とは何か・他ツール（OpenClaw / Claude Code / Codex / Cursor 等）との比較 | ✅ 翻訳済 |
| [docs/onboarding.md](docs/onboarding.md) | 初回起動オンボーディングガイド（プロバイダー選択・Base URL・ワークスペース） | ✅ 翻訳済 |
| [docs/docker.md](docs/docker.md) | Docker セットアップ完全リファレンス（1/2/3コンテナ・UID/GID・gateway） | ✅ 翻訳済 |
| [docs/remote-access.md](docs/remote-access.md) | リモートアクセス（SSH トンネル・Tailscale でスマホ利用） | ✅ 翻訳済 |
| [docs/wsl-autostart.md](docs/wsl-autostart.md) | Windows / WSL2 自動起動（WSL セッション・タスクスケジューラ） | ✅ 翻訳済 |
| [docs/supervisor.md](docs/supervisor.md) | プロセススーパーバイザ運用（launchd / systemd / supervisord） | ✅ 翻訳済 |
| [docs/advanced-chat-setup.md](docs/advanced-chat-setup.md) | 高度なチャット設定（prefill リコール・Gateway 経由チャット） | ✅ 翻訳済 |
| [docs/workspace-git.md](docs/workspace-git.md) | ワークスペース Git コントロール（信頼モデル・有効化フラグ） | ✅ 翻訳済 |
| [docs/troubleshooting.md](docs/troubleshooting.md) | トラブルシューティング（AIAgent not available・Response interrupted 等） | ✅ 翻訳済 |
| [docs/UIUX-GUIDE.md](docs/UIUX-GUIDE.md) | UI/UX ガイド（コントリビューター向け設計原則） | ✅ 翻訳済 |

### 開発者向け・設計ドキュメント（`docs/`）

| ファイル | 内容 | 状態 |
|---|---|---|
| [docs/onboarding-agent-checklist.md](docs/onboarding-agent-checklist.md) | AI アシスタント向けオンボーディング手順チェックリスト | ✅ 翻訳済 |
| [docs/CONTRACTS.md](docs/CONTRACTS.md) | プロジェクトコントラクト索引・PR 準備チェックリスト | ✅ 翻訳済 |
| [docs/EXTENSIONS.md](docs/EXTENSIONS.md) | WebUI 拡張の注入・セキュリティモデル | ✅ 翻訳済 |
| [docs/ISSUES.md](docs/ISSUES.md) | 上流 issue の根本原因分析 | ✅ 翻訳済 |
| [docs/rfcs/README.md](docs/rfcs/README.md) | RFC 規約・索引 | ✅ 翻訳済 |
| [docs/rfcs/agent-source-boundary.md](docs/rfcs/agent-source-boundary.md) | エージェントソース境界・API 分離 | ✅ 翻訳済 |
| [docs/rfcs/canonical-session-resolution.md](docs/rfcs/canonical-session-resolution.md) | 正典セッション解決コントラクト | ✅ 翻訳済 |
| [docs/rfcs/turn-journal.md](docs/rfcs/turn-journal.md) | クラッシュ安全なターンジャーナル | ✅ 翻訳済 |
| [docs/rfcs/webui-run-state-consistency-contract.md](docs/rfcs/webui-run-state-consistency-contract.md) | run 状態一貫性コントラクト | ✅ 翻訳済 |
| [docs/rfcs/live-to-final-assistant-replies.md](docs/rfcs/live-to-final-assistant-replies.md) | 長時間応答ライフサイクル | ✅ 翻訳済 |
| [docs/rfcs/hermes-run-adapter-contract.md](docs/rfcs/hermes-run-adapter-contract.md) | Run アダプタコントラクト・移行ゲート（原文 1138 行） | ✅ 翻訳済 |

### ルート主要文書（`root-docs/`）

| ファイル | 内容 | 状態 |
|---|---|---|
| [root-docs/README.md](root-docs/README.md) | プロジェクト概要・メインマニュアル（機能一覧・設定・Docker・コントリビューター。原文 723 行） | ✅ 翻訳済 |
| [root-docs/THEMES.md](root-docs/THEMES.md) | テーマ＋スキンシステム・カスタムスキン作成 | ✅ 翻訳済 |
| [root-docs/DESIGN.md](root-docs/DESIGN.md) | デザイントークンと calm-console の方向性 | ✅ 翻訳済 |
| [root-docs/AGENTS.md](root-docs/AGENTS.md) | AI アシスタント向け作業指示 | ✅ 翻訳済 |
| [root-docs/ARCHITECTURE.md](root-docs/ARCHITECTURE.md) | アーキテクチャ詳細・全APIエンドポイント・ADR・スプリントログ（原文 1658 行、全18セクション） | ✅ 翻訳済 |
| [root-docs/CONTRIBUTING.md](root-docs/CONTRIBUTING.md) | コントリビュート方針・PR の書き方 | ✅ 翻訳済 |
| [root-docs/ROADMAP.md](root-docs/ROADMAP.md) | 機能同等性チェックリスト・今後の作業・スプリント履歴 | ✅ 翻訳済 |
| [root-docs/SPRINTS.md](root-docs/SPRINTS.md) | スプリント計画・運用原則 | ✅ 翻訳済 |
| [root-docs/BUGS.md](root-docs/BUGS.md) | バグバックログ・既知の制限 | ✅ 翻訳済 |
| [root-docs/CONTRIBUTORS.md](root-docs/CONTRIBUTORS.md) | コントリビューター一覧・クレジット | ✅ 翻訳済 |
| [root-docs/TESTING.md](root-docs/TESTING.md) | 手動ブラウザテスト計画・自動カバレッジ（原文 1932 行、全37セクション＋Sprint別） | ✅ 翻訳済 |

> ✅ = 翻訳済 / ⏳ = 後続バッチで対応予定。状態は本セッション時点。

---

## 読む順番の目安

1. **まず概要**: [why-hermes.md](docs/why-hermes.md) で Hermes が何か・自分に合うかを把握。
2. **導入**: [onboarding.md](docs/onboarding.md)（ローカル）または [docker.md](docs/docker.md)（コンテナ）でセットアップ。
3. **外部からアクセス**: [remote-access.md](docs/remote-access.md)（VPS / スマホ）。
4. **常時稼働**: [supervisor.md](docs/supervisor.md)（Linux/macOS）または [wsl-autostart.md](docs/wsl-autostart.md)（Windows）。
5. **困ったら**: [troubleshooting.md](docs/troubleshooting.md)。

---

## 元リポジトリ

https://github.com/nesquena/hermes-webui
