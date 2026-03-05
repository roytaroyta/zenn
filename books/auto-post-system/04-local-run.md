---
title: "04 ローカル実行ガイド"
---

ローカルでの実行手順、デバッグ方法、よく使うコマンドをまとめます。

基本コマンド:

```bash
npm ci
npm run build
npm start
```

デバッグ:

- `ts-node` を使った直接実行
- 環境変数の切り替え（.env ファイル + dotenv）
- ログレベルの設定方法

dry-run の実行例:

- Canary ワークフローで使われる `dry_run=true` オプションに相当するローカルフラグの使い方を記載

トラブルシュート:

- よくあるエラーと対処法（依存関係の不一致、環境変数の不足）

この章ではローカルでの安全な確認とデバッグの具体手順、便利なコマンド例を紹介します。

1. 前提

- `GEMINI_API_KEY` や `BLUESKY_TOKEN` などの環境変数を `.env` に用意（本番キーは使わない）

```text
GEMINI_API_KEY=xxx
BLUESKY_TOKEN=xxx
POST_STORAGE_PATH=./data/posted.json
DRY_RUN=true
```

2. インストールとビルド

```bash
npm ci
npm run build
```

3. 生成のみ（投稿しない） — dry-run

```bash
# DRY_RUN 環境変数を指定して生成だけ行う
DRY_RUN=true npm run generate -- --topic="今日の技術メモ"

# または既定のスクリプトで dry-run フラグを渡す
npm run generate -- --dry-run
```

4. 生成→投稿（確認用）

実運用で `DRY_RUN=false` にして投稿する前に、必ず Canary アカウントやステージングアカウントでテストしてください。

```bash
DRY_RUN=false npm run generate-and-post -- --topic="テスト投稿"
```

5. 開発時の高速再起動

```bash
npm run dev
# (例: ts-node-dev を使う場合)
```

6. ログと出力の確認

- 実行ログは `./logs` に構造化 JSON で出力することを推奨します。
- 主要イベント（生成開始/生成完了/投稿成功/投稿失敗）をログに出すとデバッグが楽になります。

7. CI ワークフローをローカルで試す

- `nektos/act` を使うと、簡易的に GitHub Actions をローカルで再現できます（ただし完全一致は期待できません）。

```bash
# act の実行例（事前に act をインストール）
act -j generate-and-post --env-file .env
```

8. VS Code でのデバッグ（例）

`.vscode/launch.json` の一例:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "node",
      "request": "launch",
      "name": "Debug TS",
      "program": "${workspaceFolder}/src/index.ts",
      "preLaunchTask": "tsc: build - tsconfig.json",
      "outFiles": ["${workspaceFolder}/dist/**/*.js"]
    }
  ]
}
```

9. よくあるトラブルと対処例

- 依存関係の不一致: `node_modules` を削除して `npm ci` を再実行する
- 環境変数不足: 実行前に `printenv` や小さなチェックスクリプトで必須値の有無を検証する
- API のレート制限: 生成 API はリトライとバックオフを実装して再試行する

次の章では、投稿履歴の管理と重複検出の方法を詳述します。
