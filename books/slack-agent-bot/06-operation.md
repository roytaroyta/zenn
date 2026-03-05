---
title: "第6章：おわりに — 運用・安全策・拡張のヒント"
---

Bot が動くようになったら、次は**安定して使い続けるための運用**と、さらに便利にするための**拡張アイデア**を考えましょう。

---

## 安定稼働のための運用ヒント

### Git で必ずコミットしてから使う

`--yolo` モードは確認なしにファイルを上書きします。**作業前かつ定期的に Git にコミット**しておく習慣が必須です。

```bash
git add -A && git commit -m "$(date '+%Y-%m-%d %H:%M') 作業前スナップショット"
```

Gemini の変更が意図しないものだった場合も、`git diff` や `git checkout` ですぐに戻せます。

### Bot の自動起動（pm2）

Mac を再起動したときも Bot が自動で起動するように設定しておきます。

```bash
# pm2 でプロセス管理
npm install -g pm2
pm2 start "npx ts-node index.ts" --name slack-agent-bot
pm2 startup   # 起動スクリプトを登録
pm2 save      # 現在の設定を保存
```

`pm2 logs` でリアルタイムのログを確認できます。

### Expo の起動は手動でもよい

本書のコードでは `run` コマンドで Bot から Expo を起動しますが、常に Expo を起動済みにしておきたい場合は、Mac 側で直接起動しておく方がシンプルです。

```bash
REACT_NATIVE_PACKAGER_HOSTNAME=100.xxx.xxx.xxx npx expo start
```

Bot からの Expo 制御は「あると便利」な機能であり、なくても AI によるコード改修は動きます。

---

## よくあるトラブルと対処法

| 症状                         | 原因                          | 対処                                               |
| ---------------------------- | ----------------------------- | -------------------------------------------------- |
| Bot が応答しない             | Bot が落ちている              | `pm2 status` で確認、`pm2 restart slack-agent-bot` |
| Gemini がタイムアウトする    | ファイルが多い / モデルが重い | 軽量モデル（`fla`）に切り替えてみる                |
| Expo Go が繋がらない         | Tailscale がオフ              | スマホ側の Tailscale をオンにする                  |
| Slack がリトライを送ってくる | 処理が長すぎる                | `isProcessing` のログで状況確認                    |
| コードが意図せず壊れた       | Gemini の誤った書き換え       | `git checkout` で直前の状態に戻す                  |

---

## 拡張アイデア

### 複数人での利用

ユーザーが複数いる場合は、`isProcessing` をユーザーごとに管理したり、キューイングを実装したりすることで競合を防げます。現状の実装はシングルユーザーを想定しています。

### 指示キューの実装

複数の指示を連続して送れるよう、キューを持たせ順番に処理する実装も有効です。

```typescript
const taskQueue: string[] = [];
// キューに積んで順番に処理する...
```

### 実行前の承認ワークフロー

Slack のインタラクティブコンポーネント（ボタン）を使って、Bot が「確認してから実行しますか？」という形式でユーザーに確認を求める設計も可能です。

```
Bot: 「ログイン画面の背景を暗くする」を実行します。よろしいですか？
     [実行する] [キャンセル]
```

### Web ダッシュボード

複数のプロジェクトや実行ログを視覚的に確認したい場合は、Express や Next.js で簡易ダッシュボードを作り、Bot と連携させることも検討できます。

### 対応プロジェクトの追加

`.env` と `PROJECTS` オブジェクトに追記するだけで、管理できるプロジェクトを増やせます。

```env
PATH_PROJECT_C=/Users/your-user/Develop/react-native/project-c
```

```typescript
const PROJECTS = {
  a: { name: "プロジェクト A", path: process.env.PATH_PROJECT_A! },
  b: { name: "プロジェクト B", path: process.env.PATH_PROJECT_B! },
  c: { name: "プロジェクト C", path: process.env.PATH_PROJECT_C! }, // ← 追加
};
```

---

## セキュリティ上の注意

本書の Bot は個人利用・信頼できる Slack ワークスペースでの利用を前提としています。他者が利用する環境では以下を検討してください。

- **Bot を使えるユーザーの制限**（ユーザー ID チェック）
- **コマンドインジェクション対策の強化**（`execFile` の使用など）
- **誤ってパスやIPを出力しないログ設計**

---

## おわりに

この環境が完成すれば、開発のパラダイムが変わります。

ベッドで寝転びながら、あるいは移動中の電車の中から、スマホの Slack で「ここの余白を少し詰めて」とメッセージを打つだけ。数秒後には目の前のスマホの画面が自動でリロードされ、完成した UI がそこにあります。

**「Google AI Pro」という強力なサブスクリプションを、単なるチャット UI での会話だけでなく、システムに組み込んだ「自分だけのエンジニアリング・パートナー」としてフル活用する**この手法。

ぜひご自身の環境でも構築し、この体験を味わってみてください。

---

本書で参照したソフトウェアやサービス

- [Bolt for JavaScript](https://slack.dev/bolt-js/) — Slack 公式の Bot フレームワーク
- [Gemini CLI](https://github.com/google-gemini/gemini-cli) — Google 公式の Gemini CLI
- [Tailscale](https://tailscale.com) — サーバーレスのメッシュ VPN
- [Expo](https://expo.dev) — React Native の開発プラットフォーム
- [pm2](https://pm2.keymetrics.io) — Node.js プロセスマネージャー
