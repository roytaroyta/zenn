---
title: "01 セットアップ — TypeScriptで自動投稿システム構築"
---

対象: 開発者・運用担当者

前提:

- Node.js 20.x を利用することを想定
- GitHub Actions を用いた自動実行

手順（簡易）:

1. リポジトリをクローン
2. Node.js をインストール
3. 依存関係をインストール: `npm ci`
4. ビルド: `npm run build`
5. 動作確認: `npm start`（または `node dist/index.js`）

注意点:

- 開発環境での Node バージョン管理（nvm / asdf）を推奨
- CI でのキャッシュ戦略や依存バージョン固定の方針を記載する

各セクション（詳細）:

- ソース構成の説明
- 必要なツール (node, npm, git)
- ローカルでのデバッグ方法
- 本番へのデプロイ手順（最低限）

## 本（book）を管理するための `config.yaml`

Zenn CLI で本（book）を管理する際、`books/<book-slug>/config.yaml` は必須です。以下は記述例と各フィールドの説明です。

例:

```yaml
title: "本のタイトル"
summary: "本の紹介文"
topics: ["markdown", "zenn"]
published: true
price: 0
chapters:
	- introduction
	- setup
	- workflows
toc_depth: 2
```

フィールド説明:

- `title`: 本の表示タイトル
- `summary`: 本の紹介文（有料本でも公開されます）
- `topics`: トピック（タグ）を最大 5 つまで指定
- `published`: `true` で公開、`false` で下書き
- `price`: 0（無料）または 200〜5000 の間で 100 円単位（有料の場合はプライベートリポジトリ推奨）
- `chapters`: 表示順に並べるチャプターの slug（拡張子 `.md` は不要）
- `toc_depth`: 目次に表示する見出しの深さ（0〜3）

注意事項:

- `chapters` に記載されていないチャプターは zenn.dev に同期されません。順序は `chapters` に依存します。
- `price` を有料にする場合は運用方針（リポジトリの公開設定、販売ポリシー）を確認してください。
- カバー画像は `cover.png` または `cover.jpeg` を同フォルダに置くと良いです（推奨サイズ 500x700px）。

簡単な確認手順:

- ローカルでプレビュー: `npx zenn preview` を使って表示を確認
- 変更をコミットして GitHub にプッシュすると Zenn 側で同期が走ります
