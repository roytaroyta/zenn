---
title: "06 モニタリングとログ"
---

運用監視、ログの保存、障害時の通知フローについてまとめます。

項目例:

- ログの出力形式（JSON 推奨）
- 保存先（GitHub Actions のアーティファクト、外部ログストレージ）
- アラート: Slack / Email / PagerDuty
- 再実行・リトライポリシー

短期でやること:

- エラーログの抽出方法と確認手順を作る
- 重要な失敗を Slack へ通知するワークフローを作成

生成と投稿のワークフローを安定させるには、ログとモニタリングが不可欠です。ここでは実践的な設計例と実装スニペットを示します。

ログ出力の設計

- フォーマット: 構造化 JSON ログを推奨（機械処理しやすい）
- 含めるべきフィールド: `timestamp`, `level`, `message`, `context`（user/topic/runId）, `traceId` 等
- ライブラリ: `pino`（高速）, `winston` 等

簡易的な `logger.ts`（pino）:

```ts
import pino from "pino";

const logger = pino({
  level: process.env.LOG_LEVEL ?? "info",
  base: { service: "auto-post" },
  timestamp: pino.stdTimeFunctions.isoTime,
});

export default logger;
```

トレーサビリティ

- 各実行に `runId`（UUID）を割り振り、生成→投稿まで同じ ID を引き回すことで相関関係が追跡しやすくなります。
- 分散トレーシングも導入可能（OpenTelemetry）

メトリクス

- 収集例: 生成リクエスト数、生成成功率、平均生成時間、投稿成功率
- エクスポート先: Prometheus（アプリで `/metrics` を公開）や Datadog

エラー監視と通知

- Sentry を使って例外を集中管理する（ソースマップも設定）
- 重大エラー発生時は Slack や PagerDuty に通知する

GitHub Actions での通知例（Slack）:

```yaml
- name: Notify Slack on failure
	if: failure()
	uses: 8398a7/action-slack@v3
	with:
		status: ${{ job.status }}
	env:
		SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

ログの保管とアーカイブ

- 短期: GitHub Actions のアーティファクトを利用
- 長期: S3 / Cloud Storage に定期的にアップロードし、期間を決めて削除

運用フロー（例）

1. 定期実行の結果をログ/アーティファクトに残す
2. 異常は Sentry と Slack に流す
3. 定期レポート（週次）で重要な指標を確認
4. 異常時は Runbook に従って切り分け（手順を文書化）

Runbook に盛り込むべき項目

- すぐに確認するコマンドとログパス
- よくある失敗ケースと初動対応
- 連絡先（オンコール、オーナー）

次の章では、Gemini に対するプロンプト設計の実践的手法を説明します。
