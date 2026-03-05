---
title: "03 GitHub Actions ワークフロー"
---

この章では、GitHub Actions を使った自動投稿ワークフローの設計と運用上の注意点をまとめます。主に「定期投稿（cron）」と「手動サニティ（canary）」の 2 種類を想定します。

設計上のポイント

- 安全第一: `DRY_RUN` フラグで実際の投稿を分離する
- 承認付きの本番環境（GitHub Environments）を使い、誤実行を防ぐ
- 同時実行防止: `concurrency` を設定して重複投稿を防止する
- ロギングとアーティファクト保存（実行結果のトレース）

サンプル: 定期投稿ワークフロー（簡易）

```yaml
name: Auto Post (cron)
on:
	schedule:
		- cron: '0 9 * * *' # 毎日 9:00 UTC
	workflow_dispatch:
jobs:
	generate-and-post:
		runs-on: ubuntu-latest
		concurrency:
			group: auto-post
			cancel-in-progress: true
		environment: production
		steps:
			- uses: actions/checkout@v4
			- name: Setup Node
				uses: actions/setup-node@v4
				with:
					node-version: '20'
			- name: Install dependencies
				run: npm ci
			- name: Run tests
				run: npm test
			- name: Generate and post (dry-run by default)
				env:
					GEMINI_API_KEY: ${{ secrets.GEMINI_API_KEY }}
					BLUESKY_TOKEN: ${{ secrets.BLUESKY_TOKEN }}
					DRY_RUN: true
				run: npm run generate-and-post -- --dry-run
			- name: Upload run artifacts
				uses: actions/upload-artifact@v4
				with:
					name: auto-post-output
					path: ./logs/last-run.json
```

ポイント解説

- `environment: production` を使う場合は、`Environments` 側で保護ルール（承認者）を設定できます。
- `concurrency` で同時実行を防ぐと、重複投稿やストレージ競合を避けられます。
- 最初は `DRY_RUN=true` で動かし、安定後に本番フラグを切り替えます。

Canary（手動サニティ）ワークフロー

- 本番アカウントで実行する前に、ステージング向けのアカウントや `dry_run=false` を使った限定実行で確認します。
- `workflow_dispatch` の `inputs` を使って `dry_run`, `topic`, `force` などのパラメータを受け取れます。

手動実行の例（入力あり）:

```yaml
on:
	workflow_dispatch:
		inputs:
			dry_run:
				description: 'Dry run (true/false)'
				required: true
				default: 'true'
			topic:
				description: 'Topic to generate'
				required: false
```

運用上の検証項目

- Secrets の正当性: 実行前に `env` チェックステップを入れる
- レート制限: 生成 API の呼び出しがレート制限に引っかからないか
- エラーハンドリング: エラー時に Slack 通知や Issue 化する
- ロールバック: 誤投稿が発生した場合の対応手順をドキュメント化

段階的な導入戦略

1. ローカルで `DRY_RUN=true` を徹底して確認
2. Canary を `workflow_dispatch` で手動実行（ステージングアカウント）
3. cron を低頻度（週 1 回など）で有効化
4. 安定後に頻度を上げ、モニタリングを強化する

次の章では、ローカル実行の具体的なコマンドとデバッグ手順を説明します。
