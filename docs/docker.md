# Hermes WebUI — Docker セットアップガイド

これは Docker の包括的なリファレンスです。5分のクイックスタートは [README の Docker セクション](../README.md#docker) を参照してください。

## TL;DR — どれか1つを選ぶ

| 構成 | 使う場面 | ファイル |
|---|---|---|
| **シングルコンテナ**（推奨） | とにかくチャットを動かしたい。WebUI がエージェントをインプロセスで実行。 | `docker-compose.yml` |
| **2コンテナ** | gateway（CLI/Telegram/cron）とチャット UI を隔離したい。 | `docker-compose.two-container.yml` |
| **3コンテナ** | 2コンテナ＋監視用ダッシュボード。 | `docker-compose.three-container.yml` |
| **オールインワンイメージ**（コミュニティ fork — サードパーティ製、当方非保守） | Podman 3.4 / マルチアーキ / supervisord 風を好む。 | [sunnysktsang/hermes-suite](https://github.com/sunnysktsang/hermes-suite) — 元の議論は [#1399](https://github.com/nesquena/hermes-webui/issues/1399) を参照 |

何かが動かなくなったら、**まずシングルコンテナ構成から始めてください**。最も単純な経路で、権限/UID/パス不一致の問題のほとんどを構造的に解消します。

## 本番イメージのセキュリティモデル

本番 Docker イメージは、通常のシングルテナントなコンテナ脅威モデルに合わせてハードニングされています。Hermes WebUI は、1人のオペレーターがコンテナ、マウントした Hermes home、ワークスペースを制御することを前提とします。イメージは `sudo` を **インストールせず**、ランタイムユーザーを sudo グループに追加せず、`NOPASSWD` 昇格も付与しません。エージェント/ツールプロセスが `hermeswebui` としてシェルを得ても、パスワードなしの sudo コマンドで root になれないようにすべきです。

エントリポイントは、狭い init フェーズのために依然として `root` で開始します。Docker の bind mount は、アプリが `~/.hermes`、`/workspace`、`/app`、`/uv_cache` を読めるようになる前に UID/GID 整合や所有権の準備をしばしば必要とするためです。そのセットアップ後、`docker_init.bash` は自身を非特権の `hermeswebui` ユーザーとして再 exec し、そこでサーバーを起動します。`/tmp/hermeswebui_init` 配下の init スクラッチファイルは所有者のみ（ディレクトリ `0700`、ファイル `0600`）で、誰でも書き込み可能ではありません。

マルチテナントや敵対的コンテナ環境向けには、独自のランタイムユーザー、マウントポリシー、スーパーバイザ前提で再ビルドしてください。パッケージマネージャの利便性が必要な開発イメージは、パスワードなし sudo を本番に再導入するのではなく、開発専用 Dockerfile にそれらのツールを追加してください。

## 5分クイックスタート（シングルコンテナ）

```bash
git clone https://github.com/nesquena/hermes-webui
cd hermes-webui
cp .env.docker.example .env
# 必要なら .env を編集（Linux のほとんどのユーザーは省略可）
docker compose up -d
open http://localhost:8787
```

実運用の個人 Docker インストールはこれだけです。既存の `~/.hermes` ディレクトリがマウントされ、`~/workspace` が参照可能になり、WebUI はマウントしたボリュームから UID/GID を自動検出します。

シングルコンテナ構成は WebUI のみを実行します。cron ジョブを作成し、Tasks パネルから手動実行できます。Docker では、スケジュールジョブは、あなたが不在の間に動き続ける Hermes gateway デーモンを必要とします。System Settings に `Gateway not configured` と表示される場合は、`docker-compose.two-container.yml` か `docker-compose.three-container.yml` を使うか、オフラインのスケジュール実行に頼る前に `hermes gateway` を別途実行してください。完全な背景と検証手順は後述の [スケジュールジョブと gateway デーモン](#スケジュールジョブと-gateway-デーモン) を参照してください。

トラブルシューティング、再インストール、オンボーディング再現試験では、実状態をテストしたい意図がない限り、実運用の `~/.hermes` をマウントしないでください。隔離した Hermes home を使い、代わりに [`docs/onboarding-agent-checklist.md`](onboarding-agent-checklist.md) に従ってください。

> **Linux 注記**: Hermes home を所有するユーザーで Compose を実行してください。`sudo docker compose up -d` は Compose に `${HOME}` を `/root` として展開させることがあり、デフォルトの `${HOME}/.hermes` bind mount が、あなたのユーザーの実 Hermes ディレクトリではなく `/root/.hermes` になってしまいます。ユーザーを `docker グループ` に追加して `docker compose up -d` を実行する方を推奨します。一度きりの root 実行で呼び出し元環境を保持する必要がある場合は `sudo -E docker compose up -d` を使い、先に `docker compose config` でレンダリングされたマウントを検証してください。

## スケジュールジョブと gateway デーモン

**症状**: Tasks パネルで作成した cron ジョブが一度も発火しない。System Settings または Tasks に次が表示される:

- オレンジの "Gateway not configured"、または
- ランタイムメタデータが古いとき赤の "Gateway metadata stale"、または
- WebUI に gateway URL が設定されているがそのヘルスエンドポイントに到達できないとき赤の "Gateway endpoint not reachable"。

**原因**: スケジュールされた cron tick は WebUI 自身では駆動されません。gateway デーモンが60秒ごとにスケジューラを tick します。1つも動いていなければ、スケジュールジョブは待機したままです。"Run now" / "Trigger" ボタンは WebUI がインプロセスで処理するため、引き続き動作します。

古い gateway ビルド、またはデーモンが別コンテナで動く場合、`gateway_state.json` が古くなり、デーモンが起動していても WebUI が信頼を失うことがあります。特に base URL のみ設定されている（例: `HERMES_WEBUI_GATEWAY_BASE_URL`）うえ、ローカルのデーモン状態ファイルがリフレッシュされていない場合に顕著です。

**修正**: gateway コンテナを WebUI と並べて動かします。2コンテナの compose ファイルが推奨経路です:

```bash
cp .env.docker.example .env
docker compose -f docker-compose.two-container.yml up -d
```

3コンテナレイアウトはダッシュボードを追加しますが、それ以外は同じ形です。どうしてもシングルコンテナにとどまる場合、コンテナ内で `hermes gateway` を長寿命のバックグラウンドプロセスとして実行できますが、compose 分割の方が堅牢です。

**検証**: gateway が起動すると、System Settings のピルが緑になり、Tasks のバナーが消えるはずです。ホストから:

```bash
export GATEWAY_BASE_URL="${HERMES_API_URL:-${HERMES_WEBUI_GATEWAY_BASE_URL:-http://hermes:8642}}"
docker compose -f docker-compose.two-container.yml exec hermes-agent hermes gateway status
curl -sS "${GATEWAY_BASE_URL%/}/health/detailed" | jq '.gateway_state, .state'
```

compose ファイルでサービス名が異なる場合は、`docker compose -f docker-compose.two-container.yml ps` で稼働中サービスを一覧できます。コンテナ間診断では、gateway チャットモード（`HERMES_WEBUI_CHAT_BACKEND=gateway`）を使うとき WebUI 環境に `HERMES_API_URL` か `HERMES_WEBUI_GATEWAY_BASE_URL` のいずれかを設定し、WebUI を再起動してください。

参照 #2785。

## 何が問題になるか（と修正方法）

### 互換性ポリシーとバージョン固定

WebUI は現在実行中のバージョンを表示しますが、その表示自体は、あなたのエージェントリリースとのテスト済み互換性を保証するものではありません。

[#1925](https://github.com/nesquena/hermes-webui/issues/1925) と [#2491](https://github.com/nesquena/hermes-webui/issues/2491) の互換性境界の作業が着地するまで、WebUI と Hermes Agent のデプロイは **リリースのペア** として扱うべきです。WebUI リリースは対応するエージェントリリースに対してテストされており、一緒にアップグレード/固定すべきです。

`latest` を使う場合は両側で一貫して使い、固定タグと `latest` の混在を避けてください:
- 固定 WebUI タグ ＋ `hermes-agent:latest`
- `hermes-webui:latest` ＋ 固定 `hermes-agent` タグ

マルチコンテナ構成で固定ペアを動かす必要がある場合は、`docker-compose.two-container.yml`/`docker-compose.three-container.yml` で対応タグを使い、エージェントイメージをアップグレードするたびに [エージェントコンテナのアップグレード](#エージェントコンテナのアップグレード) のエージェントボリューム更新ワークフローを実施してください。

バージョン混在アップグレード後に挙動の問題が出たら、WebUI と hermes-agent の両バージョンと compose レイアウトを issue に記載してください。

### 1. 起動時の "Permission denied"

**症状**: コンテナは起動するが即クラッシュし、ログに次が出る:
```
PermissionError: [Errno 13] Permission denied: '/home/hermeswebui/.hermes/...'
```

**原因**: コンテナのユーザー（デフォルト UID 1000）が、ホストファイルが別の UID 所有のため、bind マウントしたディレクトリを読めない。

**修正**: `.env` の `UID` と `GID` をホストに合わせます:
```bash
echo "UID=$(id -u)" >> .env
echo "GID=$(id -g)" >> .env
docker compose down && docker compose up -d
```

macOS ではホスト UID は 501 から始まります。Linux では最初の対話ユーザーは通常 UID 1000 です。

> **macOS Docker Desktop**: env 修正後も UID マッピングが不調なら、**Settings → General → File sharing implementation** を VirtioFS と gRPC-FUSE の間で切り替えてみてください。実装によりホスト/コンテナ境界をまたぐ UID の保持方法が異なります。

### 2. ".env file mode 0640 → permission denied" (#1389)

**症状**: ホストの `.env` ファイルに `HERMES_HOME_MODE=0640`（または他のグループ可読モード）を設定し、コンテナが起動した後にエラーになる:
```
[security] fixed permissions on .env (0o640 -> 0600)
failed to load .env: open .env: permission denied
```

**原因**: WebUI の `fix_credential_permissions()` 起動フックがデフォルトで 0600 を強制します。クリーンインストールには正しい挙動ですが、オペレーター設定のモードと衝突します。

**修正**: `.env` に次のいずれかの環境変数を設定します:
- `HERMES_SKIP_CHMOD=1` — フィクサを完全にバイパス
- `HERMES_HOME_MODE=0640` — グループビットを許可し、world-readable のみ剥がす

どちらも `api/startup.py::fix_credential_permissions()` に文書化されています。

> ⚠️ **マルチコンテナ警告**: `HERMES_HOME_MODE` はエージェントイメージと WebUI で **意味が異なります**:
> - **WebUI**: 認証情報 *ファイル* モードのしきい値（`0640` は `.env` のグループビットを許可）
> - **Agent**: `HERMES_HOME` *ディレクトリ* モード（デフォルト `0700`）
>
> ディレクトリの `0640` には owner-execute ビットがないため、エージェントが自身の home を辿れず → 起動不能になります。マルチコンテナ構成では `HERMES_HOME_MODE=0750`（グループ traverse 可）か `0701`（x のみ）を使ってください。compose ファイルには各側の意味に合ったサービスごとのコメントがあります。

### 3. "ファイルがあるのにワークスペースが空に見える"

**症状**: WebUI は読み込まれるが `/workspace` にファイルが表示されない。

**原因**: #1 と同じ — bind マウントの UID 不一致。

**修正**: #1 と同じ — `.env` でホスト UID/GID を合わせる。

### 4. "2コンテナ構成: WebUI がエージェントソースを見つけられない" (#858)

**症状**: WebUI が起動時に次をログ出力:
```
!! WARNING: hermes-agent source not found.
!!   Looked in: /home/hermeswebui/.hermes/hermes-agent
!!              /opt/hermes
```

**原因**: エージェントのソース（エージェントコンテナ内の `/opt/hermes`）を、共有ボリューム経由で WebUI コンテナに公開する必要があります。2コンテナ compose ファイルは `hermes-agent-src` 名前付きボリュームでこれを行いますが、bind マウントを誤用するとパスが解決されません。

**修正**: `docker-compose.two-container.yml` 同梱の名前付きボリュームを使ってください。分かっている場合を除き bind マウントに置き換えないでください。エージェントコンテナはソースを `/opt/hermes` に書き込み、WebUI はそのボリュームを `/home/hermeswebui/.hermes/hermes-agent` にマウントします。

どうしても bind マウントを使う場合: ホストパスを1つ選び、それをエージェントコンテナの `/opt/hermes` と、WebUI コンテナの `/home/hermeswebui/.hermes/hermes-agent` の両方にマウントしてください。

### 5. "2コンテナ構成でツール（git, node 等）が無い" (#681)

**症状**: チャットでエージェントに `git status` を実行させると `command not found` でエラーになる。

**原因**: これは **アーキテクチャ上の仕様であり、バグではありません**。2コンテナ構成では、WebUI が起動したエージェントプロセスは、エージェントコンテナではなく **WebUI コンテナ内** で動きます。WebUI イメージは設計上 git/node を含みません（ツールホストではなく UI イメージです）。

**回避策**:
- **シングルコンテナ構成**（`docker-compose.yml`） — すべて1コンテナ内、境界なし
- **カスタム WebUI イメージ** — `Dockerfile` を拡張して必要なツールをインストール
- **統合イメージ**（[sunnysktsang/hermes-suite](https://github.com/sunnysktsang/hermes-suite)） — agent＋webui＋dashboard を1コンテナで提供するコミュニティ fork

### 6. "config.yaml が読み込まれない"

**症状**: ホストの `~/.hermes/` に `config.yaml` があるのに、WebUI が "no model configured" を表示する、またはカスタムプロバイダーを拾わない。

**原因**: ファイルが読めない（UID/GID 問題、#1 参照）か、コンテナ内の想定パスにない。

**修正**:
- 確認: `docker exec hermes-webui ls -la /home/hermeswebui/.hermes/config.yaml`
- 存在しない場合: ホストの bind マウントが誤ったディレクトリを指している。
- 存在するが読めない場合: UID/GID 修正は #1 参照。

### 7. "Podman で .hermes をコンテナ間共有できない"

**症状**: 2コンテナ構成が Docker では動くが、Podman では UID/GID をどう設定しても権限エラーで失敗する。

**原因**: Podman 3.4（Ubuntu 22.04 デフォルト）は複数コンテナにわたる `userns_mode: keep-id` のサポートが限定的で、あるコンテナが書いたファイルが他方で異なる UID に見えます。

**修正**: Podman 4+（これを修正済み）にアップグレードするか、[シングルコンテナ構成](#5分クイックスタートシングルコンテナ) を使うか、[コミュニティ製オールインワンイメージ](https://github.com/sunnysktsang/hermes-suite) を使ってください。

### 8. "localhost に設定した API base URL が Docker から失敗する" (#3012)

**症状**: プロバイダー、ローカルモデルサーバー、webhook、カスタム API がホストの `http://localhost:<port>` では動くが、Docker で動く Hermes WebUI に同じ URL を設定すると失敗する。

**原因**: コンテナ内では `localhost` は *そのコンテナ* を意味し、あなたのラップトップ/ホストではありません。WebUI プロセスは、サービスが同一コンテナ内で動いていない限り、`127.0.0.1` 経由でホストサービスに到達できません。

**修正**: Docker でホストする WebUI を、代わりにホストの gateway 名へ向けます:

- macOS/Windows の Docker Desktop: `http://host.docker.internal:<port>`
- Podman: `http://host.containers.internal:<port>`
- Linux Docker Engine: ホストサービスを Docker ブリッジアドレスで公開するか、compose サービスに host-gateway エイリアスを追加します:

```yaml
services:
  hermes-webui:
    extra_hosts:
      - "host.docker.internal:host-gateway"
```

その後 URL を `http://host.docker.internal:<port>` として設定します。また、ホストサービスがコンテナから到達可能なアドレスにバインドしていること（Docker ブリッジが届かないループバックインターフェースのみではないこと）、ホストのファイアウォールが接続を許可していることも確認してください。

## マルチコンテナアーキテクチャ

2/3コンテナ構成は、理由があってデフォルトで（bind マウントではなく）**名前付き Docker ボリューム** を使います。名前付きボリュームは UID/GID 問題を構造的に解決します。Docker がボリュームのルートディレクトリを正しい所有権で作成し、それを読み書きする全コンテナが同じファイルを見ます。ホスト側の権限設定は不要です。

```
                 ┌─────────────────────────────────┐
                 │      hermes-home (volume)       │
                 │  (config, sessions, state, ...)  │
                 └─────────────────────────────────┘
                          ↑              ↑
                          │ rw           │ rw
                          │              │
      ┌──────────────┐    │              │    ┌──────────────┐
      │ hermes-agent │────┘              └────│ hermes-webui │
      │  (port 8642) │                        │  (port 8787) │
      └──────────────┘                        └──────────────┘
              │                                       ↑
              │ rw                                    │ ro
              ↓                                       │
      ┌─────────────────────────┐                     │
      │ hermes-agent-src (vol)  │─────────────────────┘
      │ (agent's Python source) │
      └─────────────────────────┘
```

WebUI コンテナはエージェントの Python 依存を同梱しません。起動時に `uv pip install /home/hermeswebui/.hermes/hermes-agent` を実行し、共有ボリュームからそれらをインストールします。WebUI マウントは read-only で、エージェントコンテナが唯一の書き込み手です。

## エージェントコンテナのアップグレード

`hermes-agent-src` 名前付きボリュームは、最初の `up` 時にエージェントイメージの `/opt/hermes` から初期化されます。Docker は以後の `up` のたびにボリュームをそのまま再利用します — **より新しいエージェントイメージを `docker pull` した後でも** です。キャッシュされたボリュームの中身が新イメージのソースツリーをマスクするため、`nousresearch/hermes-agent:latest` を新規に `docker pull` しただけでは、新しいエージェントコード、依存、エントリポイントは得られません。

これが [#1416](https://github.com/nesquena/hermes-webui/issues/1416) の根本原因です。症状はエントリポイント欠落のように見えましたが、実際にはエントリポイントは新イメージに存在し、古い名前付きボリュームの背後に隠れていました。

エージェントイメージをクリーンにアップグレードするには、再作成前にソースボリュームを破棄します:

```bash
# 2コンテナ構成
docker compose -f docker-compose.two-container.yml down
docker volume rm <project>_hermes-agent-src
docker compose -f docker-compose.two-container.yml pull
docker compose -f docker-compose.two-container.yml up -d

# 3コンテナ構成
docker compose -f docker-compose.three-container.yml down
docker volume rm <project>_hermes-agent-src
docker compose -f docker-compose.three-container.yml pull
docker compose -f docker-compose.three-container.yml up -d
```

`<project>` は Compose プロジェクト名（デフォルトは親ディレクトリ。`docker volume ls` で確認）に置き換えてください。`hermes-home` ボリューム（config、sessions、state）はそのまま残り、`hermes-agent-src`（エージェントのインストール済み Python ソース）のみが再作成されます。

> シングルコンテナ構成（`docker-compose.yml`）は `hermes-agent-src` を使わず、このアップグレードパターンの影響を受けません。新しい WebUI イメージを pull して `docker compose up -d --force-recreate` で十分です。

## マルチコンテナ構成が隔離するもの（としないもの）

2/3コンテナ構成は、gateway とチャット UI の間に **プロセス・ネットワーク・リソースの隔離** を与えます:

- 各サービスは独自の PID 名前空間とライフサイクルを持ちます。エージェントプロセスがクラッシュしてもチャット UI を巻き込まず、逆も同様です。
- gateway API（port 8642）はエージェントサービスのみがバインドし、WebUI はバインドできません。他のコンテナは `hermes-net` Docker ネットワーク経由で gateway に到達します。
- リソース制限（`docker-compose.three-container.yml` の `deploy.resources.limits`）はサービスごとに適用され、エージェントをダッシュボードと独立に上限設定できます。
- 再起動ポリシー、ログストリーム、コンテナヘルスチェックはサービスごとにスコープされます。

マルチコンテナが **隔離しない** もの:

- **ファイルシステム境界。** 両サービスは `hermes-home`（config、sessions、state）を共有し、WebUI はエージェントのインストール済みソースを `hermes-agent-src` からマウントします。WebUI マウントは read-only（v0.51.84 以降）ですが、エージェントサービスは依然書き込みアクセスを持ち、両サービスが home ボリュームを共有します。
- **UID/GID 境界。** 両サービスはデフォルトで `${UID:-1000}` なので、一方が書いたファイルを他方が読めます。異なる UID に揃えると共有ボリュームで権限エラーになります。
- **エージェントソースに対する信頼境界。** WebUI は起動時に共有 `hermes-agent-src` ボリュームから Python 依存をインストールします。read-only マウントにより、侵害された WebUI がエージェントソースを書き換えることはできませんが、そのボリュームのコードを実行はします。

チャット UI とエージェントの間に **ファイルシステム隔離** が必要な場合（例: WebUI がエージェント状態を読むことを信頼しない）、マルチコンテナ構成では不十分です。エージェントを別ホストで動かし、gateway HTTP API 経由で WebUI を接続してください。どんな境界も不要なら、シングルコンテナ構成の方が単純です。

直接のソースマウントは互換性のための橋渡しであり、長期的な API コントラクトではありません。現在のソース/API 境界の棚卸しと分離タスクリストは、[#2453](https://github.com/nesquena/hermes-webui/issues/2453) 向けに [`docs/rfcs/agent-source-boundary.md`](rfcs/agent-source-boundary.md) にあります。compose ファイルを bind マウントでカスタマイズする場合、意図的にローカル開発をしているのでない限り、WebUI 側のエージェントソースマウントは read-only に保ってください。`docker_init.bash` はそのパスが書き込み可能なとき起動時に警告します。

## bind マウント移行（上級者向け）

既存のホスト `~/.hermes` を本当に bind マウントする必要がある場合（例: dotfiles で config を管理、非 Docker の `hermes` インストールと共有、など）:

```yaml
volumes:
  hermes-home:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /home/youruser/.hermes
  hermes-agent-src:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /opt/hermes-agent-source
```

**重要な要件**:

1. ホストディレクトリは、あなたのコンテナ UID から読める必要があります。ホストで `id -u` を実行し、`~/.hermes` がその UID 所有（またはグループビットで可読）であることを確認してください。
2. ボリュームを共有する全コンテナは、同じ UID/GID で動く必要があります。`.env` に `UID=$(id -u)` と `GID=$(id -g)` を設定してください。
3. sudo で Compose を実行する場合、`${HOME}` のデフォルトに頼らないでください。`sudo` はしばしば `$HOME` を `/root` に変えるため、`${HERMES_HOME:-${HOME}/.hermes}` が `/root/.hermes` になります。自分のユーザーで Docker を実行する方を推奨します。やむを得ない場合は `sudo -E` で絶対パスを渡し（例: `HERMES_HOME=/home/youruser/.hermes HERMES_WORKSPACE=/home/youruser/workspace sudo -E docker compose up -d`）、`docker compose config` でレンダリングされた bind マウントを確認してください。
4. ホストの `.env` がモード 0640 の場合、起動フックが 0600 を強制しないよう `HERMES_SKIP_CHMOD=1` か `HERMES_HOME_MODE=0640` を設定してください。

## リファレンス

- [`docker-compose.yml`](../docker-compose.yml) — シングルコンテナ（推奨）
- [`docker-compose.two-container.yml`](../docker-compose.two-container.yml) — agent ＋ webui
- [`docker-compose.three-container.yml`](../docker-compose.three-container.yml) — agent ＋ dashboard ＋ webui
- [`.env.docker.example`](../.env.docker.example) — 環境変数テンプレート
- [`Dockerfile`](../Dockerfile) — シングルコンテナビルド
- [`docker_init.bash`](../docker_init.bash) — コンテナエントリポイントスクリプト

## 関連 issue

- #1416 — エージェントイメージのアップグレードには `hermes-agent-src` 名前付きボリュームの削除が必要（[エージェントコンテナのアップグレード](#エージェントコンテナのアップグレード) 参照）
- #1389 — `HERMES_HOME_MODE` の上書き（v0.50.254 で修正 — エージェントが `HERMES_SKIP_CHMOD` と `HERMES_HOME_MODE` を尊重）
- #1399 — compose ファイルの UID 整合（PR #1428 ＋本ガイドで v0.50.260 にて修正）
- #3012 — ホストの `localhost` API URL が Docker コンテナから失敗（`host.docker.internal` / `host.containers.internal` を使う）
- #3006 — `sudo docker compose` がユーザーの Hermes home ではなく `/root/.hermes` をマウントし得る
- #858 — 2コンテナの `/opt/hermes` パスの混乱
- #681 — ツールがエージェントコンテナではなく WebUI コンテナで動く（アーキテクチャ上の仕様）
- #668 — マウントしたボリュームから UID/GID を自動検出
- #569 — UID/GID 検出の優先順位

ここに載っていない新しい障害モードに遭遇したら、次を添えて [issue を開いて](https://github.com/nesquena/hermes-webui/issues/new) ください:

1. 使った compose ファイル
2. `docker logs hermes-webui` のエラー
3. `docker exec hermes-webui id` の出力
4. `docker exec hermes-webui ls -la /home/hermeswebui/.hermes` の出力

---

> 本ドキュメントは hermes-webui 公式リポジトリ `docs/docker.md` の日本語訳です。コマンド・パス・環境変数・バージョン番号は原文のまま表記しています。見出しへのアンカーリンクは日本語見出しに合わせて調整しています。
