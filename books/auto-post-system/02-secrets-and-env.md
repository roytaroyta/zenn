---
title: "02 Secrets と環境変数"
---

このシステムで必要となるシークレットとその用途、設定方法をまとめます。

必須シークレット（GitHub Actions）:

- GNEWS_API_KEY — ニュース取得用 API キー
- GEMINI_API_KEY — 要約生成（Gemini）用 API キー
- TWITTER_APP_KEY — X API (旧 Twitter) 用
- TWITTER_APP_SECRET
- TWITTER_ACCESS_TOKEN
- TWITTER_ACCESS_SECRET

ローカルでの設定例:

```bash
export GNEWS_API_KEY=...
export GEMINI_API_KEY=...
export TWITTER_APP_KEY=...
# ...
```

注意事項:

- Secrets は Git 管理やログに残さないこと
- テスト用と本番用のキーは分ける
- アクセス権限の最小化（読み取り専用など）を検討

拡張:

- Vault / secret manager を用いた運用例（将来的な案）
