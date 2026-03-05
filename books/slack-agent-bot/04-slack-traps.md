---
title: "第4章：Slack Bot 開発における「2つの大きな罠」と解決策"
---

「Slack からコマンドを叩くだけ」と聞くと簡単そうですが、実運用に耐えうる Bot を作るには、必ずと言っていいほど以下の 2 つの罠にハマります。**先に知っておくことで、実装後の混乱を避けられます**。

---

## 罠1：Slack の「3秒ルール」とリトライ爆撃

### 何が起きるか

Slack のイベント API は、Bot 側から **「受け取りました（200 OK）」** の返事が **3 秒以内に来ない**と、「通信エラーかな？」と判断して**同じリクエストを最大 3 回まで再送**してきます。

Gemini がファイルの読み書きと推論を行うには 10〜30 秒かかります。そのまま素朴に作ると：

```
ユーザーが 1 回指示
 ↓
Gemini が起動（時間がかかる）
 ↓ 3秒後に Slack がリトライ
 ↓ Gemini がさらに起動（2台目）
 ↓ 再度リトライ
 ↓ Gemini がさらに起動（3台目）

→ 同じ指示を受けた Gemini が3〜4台同時にファイルを書き換え始める
→ コードが滅茶苦茶になる
```

という地獄絵図が発生します。

### 解決策：リトライの検知 ＋ 排他ロック

**① Slack リトライを検知して無視する**

リトライリクエストのヘッダーには `x-slack-retry-num` が付与されています。Bolt では `body.retry_attempt` として参照できます。

```typescript
app.event("app_mention", async ({ event, body, say }) => {
  // リトライリクエストは即座に無視する
  if (body.retry_attempt) {
    console.log("⏭️ Slackのリトライを無視しました。");
    return; // ← ここで早期リターン
  }

  // ... 以降の処理
});
```

**② isProcessing フラグで二重起動を防止する**

リトライを完全に防げない場合や、ユーザーが連続して指示を送った場合に備えて、処理中かどうかを管理するフラグを設けます。

```typescript
let isProcessing = false; // ★ グローバルな排他ロックフラグ

// ...

if (isProcessing) {
  await say("⏳ 前の指示を処理中だよ。ちょっと待ってね！");
  return;
}

isProcessing = true; // ロック開始

try {
  // Gemini の処理
} finally {
  isProcessing = false; // 必ず解除（エラー時も含む）
}
```

`finally` ブロックで必ずロックを解除することが重要です。例外が発生しても `isProcessing` が `true` のまま残ることを防ぎます。

---

## 罠2：CLI の対話プロンプトでフリーズする問題

### 何が起きるか

`gemini-cli` はデフォルトでは、ファイルを書き換える前に以下のようなプロンプトを表示して、**人間の確認を求めて処理を一時停止**します。

```
Are you sure you want to overwrite this file? (y/n/always)
```

Slack Bot の裏側でこれが起きると：

- Bot は Gemini からの応答を待ち続ける
- 誰も「`y`」を入力できない
- **Bot が永遠にフリーズしたまま**になる

### 解決策：`--yolo` フラグ

**YOLO（You Only Live Once）フラグ**を付与することで、人間の確認を待たずに Gemini が自律的にすべての操作を実行します。

```bash
gemini "ログイン画面の背景を暗くして" --yolo
```

`--yolo` フラグを付けると：

- ファイルの確認プロンプトをスキップ
- 自律的にファイルを読み込み・上書き保存
- Expo のホットリロードを即座に誘発

コードでは以下のように組み込みます。

```typescript
const command = `cd ${targetDir} && gemini "${safeText}" --yolo`;
const { stdout, stderr } = await execPromise(command);
```

:::message alert
**Git 管理は必須**

`--yolo` モードは確認なしにファイルを上書きします。予期しない破壊が起きた場合に戻せるよう、必ず **Git でバージョン管理**をしておいてください。

```bash
# 少なくとも作業前にコミットしておく
git add -A && git commit -m "gemini 作業前の状態"
```

:::

---

## 追加の実装上の注意：コマンドインジェクション対策

ユーザーが Slack でメッセージを送り、それをそのまま Shell コマンドに埋め込むと、悪意あるユーザーが任意のコマンドを実行できるリスク（コマンドインジェクション）があります。

最低限の対策として、ダブルクォートをエスケープします。

```typescript
const safeText = text.replace(/"/g, '\\"');
const command = `cd ${targetDir} && gemini "${safeText}" --yolo`;
```

さらに踏み込む場合は、`execFile` や引数の厳密なバリデーションを検討してください。本書では個人利用を前提としていますが、他の人が使う環境では追加のセキュリティ対策が必要です。

---

この 2 つの罠と解決策を押さえた上で、次の章では完成版のコード全体を丁寧に解説します。
