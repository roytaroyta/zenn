---
title: "第5章：決定版 index.ts の全貌解説"
---

これまで解説してきた「Tailscale 連携」「Slack リトライ対策」「二重起動防止」「`--yolo` モード」「モデル切り替え」をすべて組み込んだ、完成版の Bot コードを部分ごとに丁寧に解説します。

---

## 全体のコード

まずは全体像を示します。個人情報（パス・IP）はマスクしています。

```typescript
import { App } from "@slack/bolt";
import * as dotenv from "dotenv";
import { exec, spawn, ChildProcess } from "child_process";
import util from "util";

dotenv.config();
const execPromise = util.promisify(exec);

const app = new App({
  token: process.env.SLACK_BOT_TOKEN!,
  appToken: process.env.SLACK_APP_TOKEN!,
  socketMode: true,
});

// ==========================================
// 1. 設定項目
// ==========================================
const TAILSCALE_IP = process.env.TAILSCALE_IP!;

const SELECT_PROJECT = "set ";
const SELECT_MODEL = "model ";
const EXPO_START = "run";
const EXPO_STOP = "stop";
const HELP = "help";

// プロジェクト定義（パスは .env で管理）
const PROJECTS: { [key: string]: { name: string; path: string } } = {
  a: { name: "プロジェクト A", path: process.env.PATH_PROJECT_A! },
  b: { name: "プロジェクト B", path: process.env.PATH_PROJECT_B! },
};

// ==========================================
// 2. 状態管理
// ==========================================
let currentProject = "a";
let expoProcess: ChildProcess | null = null;
let isProcessing = false; // AIが作業中かどうかの排他ロック

// ==========================================
// 3. モデル管理
// ==========================================
let currentModel: string | null = null;
let currentModelAlias: string | null = null;

const MODEL_ALIASES: { [key: string]: string } = {
  fla: "gemini-2.5-flash",
  fla3: "gemini-3-flash-preview",
  pro: "gemini-3-pro-preview",
};

// ==========================================
// 4. イベントハンドラ
// ==========================================
app.event("app_mention", async ({ event, body, say }) => {
  // ① Slack リトライを無視
  if (body.retry_attempt) {
    console.log("⏭️ Slackのリトライを無視しました。");
    return;
  }

  const text = event.text.replace(/<@.+?>/, "").trim();
  const lowerText = text.toLowerCase();
  const { name: currentName, path: targetDir } = PROJECTS[currentProject]!;

  // ② ヘルプ表示
  if (lowerText === HELP) {
    await say(`📖 *利用可能なコマンド:*
• \`${EXPO_START}\`            : Expo を起動
• \`${EXPO_STOP}\`             : Expo を停止
• \`${SELECT_PROJECT}<id>\`    : プロジェクト切替 (a, b)
• \`${SELECT_MODEL}<alias>\`   : モデル切替 (fla / fla3 / pro / default)
• \`${HELP}\`                  : このヘルプを表示
• \`<指示内容>\`               : Gemini で作業を実行`);
    return;
  }

  // ③ プロジェクト切り替え
  if (lowerText.startsWith(SELECT_PROJECT)) {
    const target = lowerText.replace(SELECT_PROJECT, "").trim();
    if (PROJECTS[target]) {
      currentProject = target;
      await say(
        `🔄 プロジェクトを [${PROJECTS[currentProject]!.name}] に切り替えました。`,
      );
    } else {
      await say(
        `⚠️ [${target}] は見つかりません。(${Object.keys(PROJECTS).join(" / ")})`,
      );
    }
    return;
  }

  // ④ モデル切り替え
  if (lowerText.startsWith(SELECT_MODEL)) {
    const target = lowerText.replace(SELECT_MODEL, "").trim();
    if (MODEL_ALIASES[target]) {
      currentModel = MODEL_ALIASES[target];
      currentModelAlias = target;
      await say(`✅ モデルを *${currentModel}* に切り替えました。`);
    } else if (target === "default") {
      currentModel = null;
      currentModelAlias = null;
      await say(`🔄 モデルをデフォルトに戻しました。`);
    } else {
      currentModel = target;
      currentModelAlias = target;
      await say(`✅ モデルを *${currentModel}* に設定しました。`);
    }
    return;
  }

  // ⑤ Expo 起動（Tailscale IP でバインド）
  if (lowerText === EXPO_START) {
    if (expoProcess) expoProcess.kill();
    await say(`🚀 [${currentName}] で Expo 起動中...`);
    expoProcess = spawn("npx", ["expo", "start"], {
      cwd: targetDir,
      env: { ...process.env, REACT_NATIVE_PACKAGER_HOSTNAME: TAILSCALE_IP },
      shell: true,
    });
    await say(
      `✅ 準備完了！Expo Go で以下を開いてください：\n\`exp://${TAILSCALE_IP}:8081\``,
    );
    return;
  }

  // ⑥ Expo 停止
  if (lowerText === EXPO_STOP) {
    if (expoProcess) {
      expoProcess.kill();
      expoProcess = null;
      await say(`🛑 Expo を停止しました。`);
    }
    return;
  }

  // ⑦ Gemini エージェント実行（二重起動防止つき）
  if (isProcessing) {
    await say("⏳ 前の指示を処理中だよ。ちょっと待ってね！");
    return;
  }

  isProcessing = true; // ロック開始
  await say(`🧠 Gemini で作業中...\n「${text}」`);

  try {
    const modelArg = currentModel ? `--model ${currentModel}` : "";
    const safeText = text.replace(/"/g, '\\"'); // インジェクション対策
    const command =
      `cd ${targetDir} && gemini "${safeText}" --yolo ${modelArg}`.trim();

    const { stdout, stderr } = await execPromise(command);

    // ノイズの多い stderr（YOLO モードメッセージ等）を除去
    const removeNoise = /(YOLO mode.*|Loaded cached credentials.*)\r?\n?/gim;
    const cleanStderr = (stderr || "").replace(removeNoise, "").trim();

    let reply = `✅ 完了！スマホを確認してね。\n\n*Log:*\n\`\`\`\n${stdout}\n\`\`\``;
    if (cleanStderr) reply += `\n*Stderr:*\n\`\`\`\n${cleanStderr}\n\`\`\``;

    await say(reply);
  } catch (error: any) {
    await say(`❌ エラー:\n\`\`\`\n${error.message}\n\`\`\``);
  } finally {
    isProcessing = false; // 必ずロック解除
  }
});

// ==========================================
// 5. Bot 起動
// ==========================================
(async () => {
  await app.start();
  console.log(`⚡️ Slack Agent 起動完了！`);
})();
```

---

## .env ファイルの設定

```env
# Slack の認証情報（Slack App 設定画面から取得）
SLACK_BOT_TOKEN=xoxb-...
SLACK_APP_TOKEN=xapp-...

# Tailscale で Mac に割り当てられた IP
TAILSCALE_IP=100.xxx.xxx.xxx

# プロジェクトの絶対パス
PATH_PROJECT_A=/Users/your-user/Develop/react-native/project-a
PATH_PROJECT_B=/Users/your-user/Develop/react-native/project-b
```

---

## 各セクションの解説

### 1. 設定項目

```typescript
const SELECT_PROJECT = "set ";
const SELECT_MODEL = "model ";
const EXPO_START = "run";
const EXPO_STOP = "stop";
const HELP = "help";
```

Bot が認識するコマンドのプレフィックスを定数で管理しています。文字列の変更がここだけで完結するため、メンテナンスが楽になります。

`PROJECTS` オブジェクトで、プロジェクト名と絶対パスをペアとして定義しています。パスは `.env` から読み込むことで、コードへのパスのハードコードを避けています。

### 2. 状態管理

```typescript
let currentProject = "a";
let expoProcess: ChildProcess | null = null;
let isProcessing = false;
```

- `currentProject`：現在選択されているプロジェクトのキー
- `expoProcess`：起動している Expo サーバーのプロセス参照（`kill()` で停止するために保持）
- `isProcessing`：Gemini が作業中かどうかのフラグ

### 3. リトライ排除 ＋ 排他ロック

```typescript
if (body.retry_attempt) {
  return; // Slack リトライを無視
}
// ...
if (isProcessing) {
  await say("⏳ 前の指示を処理中だよ。ちょっと待ってね！");
  return;
}
isProcessing = true;
try {
  /* 処理 */
} finally {
  isProcessing = false;
}
```

前章の「2 つの罠」の解決策をそのまま実装しています。`finally` ブロックで確実にロックを解除します。

### 4. Expo 起動（Tailscale 対応）

```typescript
expoProcess = spawn("npx", ["expo", "start"], {
  cwd: targetDir,
  env: { ...process.env, REACT_NATIVE_PACKAGER_HOSTNAME: TAILSCALE_IP },
  shell: true,
});
```

`spawn` を使う理由は、`exec` と違ってプロセスを**バックグラウンドで継続動作**させられるためです。`exec` は実行完了を待つため、常時起動するサーバーには不向きです。

### 5. Gemini エージェント実行

```typescript
const command =
  `cd ${targetDir} && gemini "${safeText}" --yolo ${modelArg}`.trim();
const { stdout, stderr } = await execPromise(command);
```

ここでは `exec`（`execPromise`）を使って Gemini の終了を待ちます。完了後にログをまとめて Slack に返します。

`stderr` にはノイズ（YOLO モードのメッセージやキャッシュ読み込みメッセージ）が含まれることがあるため、正規表現で除去してから表示します。

---

## Slack App の作成と設定

### Slack App の作成手順（概要）

1. [api.slack.com/apps](https://api.slack.com/apps) にアクセスし、「Create New App」
2. 「From scratch」を選択し、アプリ名とワークスペースを設定
3. **Socket Mode を有効化**（App-Level Token を発行）
4. **Event Subscriptions で `app_mention` を購読**
5. **Bot Token Scopes に `app_mentions:read` と `chat:write` を追加**
6. アプリをワークスペースにインストール

### 取得するトークン

| トークン  | 取得場所                             | .env への記述     |
| --------- | ------------------------------------ | ----------------- |
| Bot Token | OAuth & Permissions > Bot Token      | `SLACK_BOT_TOKEN` |
| App Token | Basic Information > App-Level Tokens | `SLACK_APP_TOKEN` |

### Socket Mode を選ぶ理由

本構成では **Socket Mode**（WebSocket 接続）を採用しています。

- **パブリックな URL が不要**（ポート開放や ngrok が不要）
- 自宅 Mac から Slack への接続はアウトバウンドのみで完結
- ファイアウォール・ルーターの設定変更が不要

---

## 起動方法

```bash
# 依存関係をインストール
npm install

# Bot を起動
npx ts-node index.ts
```

Mac が起動した際に自動でBotも起動したい場合は、`launchd`（macOS のサービス管理）や `pm2` を使うと便利です。

```bash
# pm2 で常時起動
npm install -g pm2
pm2 start "npx ts-node index.ts" --name slack-agent-bot
pm2 startup  # 再起動後も自動で起動するよう設定
pm2 save
```

起動が確認できたら、Slack でBot に `@メンション` して `help` と送ってみてください。コマンド一覧が表示されます。

---

次の章では、安定稼働させるための運用ヒントと、このシステムをさらに拡張するためのアイデアを紹介します。
