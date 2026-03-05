---
title: "03 GitHub Actions ワークフロー"
---

このページでは `.github/workflows/` にあるワークフローの目的、入力、主要な実行フローを説明します。

主なワークフロー:

- `cron-post.yml` — 定期投稿（cron）
- `canary-post.yml` — 手動サニティチェック（Canary）

cron の設定例:

- 現在: UTC 9:00 毎日（必要に応じ変更）

Canary ワークフロー利用方法:

- Actions タブから `Canary Auto-Post` を選択
- 入力パラメータ: `dry_run`（true/false）, `run_env_check`（true/false）

検証項目:

- Secrets が正しくセットされているか
- 実行時のログの出力先
- 失敗時のリトライや通知の仕組み
