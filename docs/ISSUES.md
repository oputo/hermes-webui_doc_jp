# 上流 Issue — 根本原因分析

## #1256: ブラウザツールが "Playwright not installed" で失敗

### 根本原因
このチェックは hermes-webui ではなく **hermes-agent**（上流）にあります:

```
hermes-agent/tools/browser_tool.py → check_browser_requirements()
```

`check_browser_requirements()` は CDP（Chrome DevTools Protocol）モードを認識せず、ローカルの Playwright/Puppeteer インストールしか探しません。エージェントが CDP モード（既存ブラウザに接続）で動くと、チェックは依然失敗します。

### WebUI 側
WebUI は既に `CLI_TOOLSETS` をリクエストごとに正しく渡しています。cron/chat 設定の `enabled_toolsets` フィールドは動的で、意図通り機能します。

### 必要な修正
修正は `hermes-agent/tools/browser_tool.py` で行う必要があります:
- CDP モードが設定されているとき `check_browser_requirements()` は Playwright チェックをスキップすべき
- またはローカルブラウザ要件をバイパスする `BROWSER_MODE=cdp` 環境変数を追加

### 回避策
`CLOUD_BROWSER=true` を使うか、`browser.base_url` をリモート CDP エンドポイントに向けて設定します。これでローカル Playwright 要件をバイパスできます。

---

> 本ドキュメントは hermes-webui 公式リポジトリ `docs/ISSUES.md` の日本語訳です。Issue 番号・関数名・ファイルパス・環境変数は原文のまま表記しています。
