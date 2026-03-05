---
title: "02 Secrets と環境変数"
---

この章では、API キーやアクセストークンなどの「秘密情報」を安全に管理するための実践的な方針と、ローカル／CI での設定方法を示します。

必要なシークレット（例）

- `GNEWS_API_KEY` — ニュース取得用 API キー
- `GEMINI_API_KEY` — Gemini（生成モデル）用 API キー
- `BLUESKY_TOKEN` — Bluesky 投稿用のアクセストークン
- （既存の参考実装に基づく）X/Twitter 用のキー群や、画像アップロードサービスのキーなど

ローカルでの管理

- 開発中は `.env` を利用し、`dotenv` で読み込むのが手軽です。ただし `.env` は必ず `.gitignore` に追加してください。

例 (`.env` ファイル):

```
GEMINI_API_KEY=your_gemini_api_key
BLUESKY_TOKEN=your_bluesky_token
POST_STORAGE_PATH=./data/posted.json
DRY_RUN=true
```

Node.js 側での読み込み例:

```ts
import dotenv from "dotenv";
dotenv.config();
const apiKey = process.env.GEMINI_API_KEY;
```

GitHub Actions の Secrets

- リポジトリの `Settings > Secrets and variables > Actions` にキーを追加します。
- 本番運用では `Environments`（例: `production`）を作り、承認者レビューを必須にすることで accidental run を防げます。

ワークフローでの参照例:

```yaml
env:
	GEMINI_API_KEY: ${{ secrets.GEMINI_API_KEY }}
	BLUESKY_TOKEN: ${{ secrets.BLUESKY_TOKEN }}
```

セキュリティのベストプラクティス

- Secrets は平文でコミットしない。履歴に残っている場合は直ちにローテーションする。
- 最小権限の原則を適用する（発行する API キーが必要な操作だけ許可する）。
- 本番キーとテストキーは分離する。
- ローテーション手順を文書化しておく。

高度な秘密管理オプション

- クラウドの Secret Manager（AWS Secrets Manager / GCP Secret Manager / Azure Key Vault）を使う。
  - 利点: ローテーション、監査ログ、IAM ベースのアクセス制御
- SOPS + GitOps（暗号化されたシークレットを Git 管理）
- HashiCorp Vault を導入して Dynamic Secrets を利用する

選択の指針

- 小規模プロジェクト: `.env`（ローカル） + GitHub Secrets（CI）
- 成長する運用: Secret Manager へ移行、環境ごとのアクセス制御とローテーション自動化

ログや例外での取り扱い

- エラーログに秘密が出力されないようにテンプレート化する。
- 例外やレスポンスのダンプを行う際は、マスク処理を行う（トークンの頭/末尾のみ表示する等）。

次の章では、これらの秘密を使って安全に GitHub Actions を構成する方法を説明します。
