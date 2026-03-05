---
title: "第2章：なぜ Gemini CLI なのか？ — Google AI Pro のフル活用"
---

AIにコードを書かせるツール（Cursor、Claude Code など）は多数存在しますが、本構成において `gemini-cli` を採用した理由は明確です。

**「Google AI Pro（Gemini Advanced）のサブスクリプションのパワーを、ターミナルから直接引き出せるから」**

## API キーが不要な「認証ログイン」という選択肢

通常、自作プログラムから AI を動かすには API キーを発行し、従量課金や無料枠の制限を気にしながら使う必要があります。

しかし `gemini-cli` には、Google アカウントを用いた **OAuth ログイン認証機能** が備わっています。

### ログイン手順

Mac のターミナルで以下を実行するだけです。

```bash
gemini login
```

ブラウザが立ち上がるので、Google AI Pro（Gemini Advanced）を契約している Google アカウントでログインを完了させます。

たったこれだけで：

- API キーの発行が不要
- `.env` への API キー記述が不要
- 従量課金の管理が不要
- 月額プランに紐づいた最新の高性能モデルを利用可能

という状態になります。

## gemini-cli のインストール

Node.js が入っていれば `npm` でインストールできます。

```bash
npm install -g @google/gemini-cli
```

インストール後、ログインを実行します。

```bash
gemini login
```

ターミナル上で以下のように動作確認できます。

```bash
gemini "TypeScriptで Hello World を書いて"
```

## エージェントとしての動作

`gemini-cli` がただの「回答を返す AI」と異なる点は、**プロジェクトのファイルを自律的に読み込み・書き換える「エージェント」として動作できる**ことです。

特定のディレクトリで実行すると、Gemini はそのディレクトリ内のファイルを参照し、指示に沿ってコードを修正して上書き保存します。

```bash
cd /path/to/your-project
gemini "ログイン画面の背景色を #1a1a2e に変更して" --yolo
```

`--yolo` フラグについては後の章で詳しく説明しますが、これが「人間の確認待ち」を省略して完全自動化するための重要なフラグです。

## モデルの切り替え

`gemini-cli` はフラグで使用モデルを切り替えられます。

```bash
# Gemini 2.5 Flash を使う（高速・コスト効率重視）
gemini "指示" --model gemini-2.5-flash

# デフォルト（ログインアカウントのプランに応じた最新モデル）
gemini "指示"
```

本書の Bot 実装では、Slack からモデルを動的に切り替えるコマンドも用意しています。

```
# Slack でモデルを切り替える例
@bot model fla      → Gemini 2.5 Flash に切り替え
@bot model pro      → Gemini 3 Pro Preview に切り替え
@bot model default  → デフォルトに戻す
```

## 注意点

- `gemini login` によるログイン情報は、Mac のキーチェーンや設定ファイルに保存されます
- Google AI Pro のサブスクリプションが有効であることが前提です
- 利用規約の変更や API 仕様の変更によって挙動が変わる可能性があります。公式ドキュメントを定期的に確認してください

次の章では、外出先のスマホから Expo の開発画面をリアルタイム確認するための Tailscale 設定を解説します。
