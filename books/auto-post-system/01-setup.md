---
title: "01 セットアップ — TypeScriptで自動投稿システム構築"
---

この章ではローカル開発環境の準備からビルド・実行まで、実務で使える手順を示します。

前提

- 推奨 Node.js バージョン: 18.x または 20.x（プロジェクトの `engines` を確認）
- TypeScript と npm/yarn/pnpm のいずれかに慣れていること

1. リポジトリの取得

```bash
git clone <your-repo-url>
cd bluesky-post-bot
```

2. Node.js の準備（バージョン管理推奨）

nvm を使う例:

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.4/install.sh | bash
nvm install 20
nvm use 20
```

```bash
asdf plugin-add nodejs https://github.com/asdf-vm/asdf-nodejs.git
asdf install nodejs 20.0.0
asdf global nodejs 20.0.0
```

3. 依存関係のインストール

CI と同じ再現性を得るために `package-lock.json` / `pnpm-lock.yaml` を使い、以下を実行します:

```bash
npm ci
# または
pnpm install --frozen-lockfile
# または
yarn install --frozen-lockfile
```

4. 開発用スクリプト（推奨）

package.json に以下のようなスクリプトを用意しておくと運用が楽です。

```json
"scripts": {
	"build": "tsc -p tsconfig.json",
	"start": "node dist/index.js",
	"dev": "ts-node-dev --respawn src/index.ts",
	"lint": "eslint src --ext .ts",
	"test": "jest"
}
```

5. ビルドと実行

```bash
npm run build
npm start
```

開発中は `npm run dev` を使って TypeScript の変更を即時反映しながら動かすと便利です。

6. ローカルでの環境変数管理

`.env` を使う場合は開発専用にし、実運用では秘密は Git に含めないでください。例:

```
GEMINI_API_KEY=your_gemini_api_key
BLUESKY_TOKEN=your_bluesky_token
POST_STORAGE_PATH=./data/posted.json
DRY_RUN=true
```

開発時には `dotenv` を利用して読み込むか、`env` ユーティリティを用いて `process.env` を整理してください。

7. CI の準備メモ（GitHub Actions）

- `actions/setup-node` で Node バージョンを固定
- `actions/cache` で npm/pnpm のキャッシュを活用
- `npm ci` を使ってロックファイルに基づく再現性あるインストール

8. 追加の注意点

- ローカルと CI で同一の Node バージョンを使うこと（`.nvmrc` / `.tool-versions` をリポジトリに含める）
- 依存関係は定期的にアップデートし、脆弱性スキャンを CI に組み込む
- 本番実行では `DRY_RUN` を `false` に切り替えるための安全措置（手動確認や承認ワークフロー）を設ける

この章の次は `Secrets と環境変数` の章で、API キーやトークンの安全な管理方法について詳述します。
