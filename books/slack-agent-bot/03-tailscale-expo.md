---
title: "第3章：Tailscale × Expo による「どこでもライブプレビュー」"
---

スマホの Expo Go アプリで開発中の画面をプレビューする場合、通常は「Mac と同じ Wi-Fi に接続していること」が条件です。

しかし外出先からこれを実現するには、**Tailscale** というメッシュ VPN を使います。

## Tailscale とは

Tailscale は、複数のデバイスを**同じプライベートネットワーク**に見せかけてくれる無料の VPN サービスです。

- **サーバー不要**：ピアツーピアで直接通信
- **設定が非常に簡単**：アプリをインストールして同じアカウントにログインするだけ
- **個人利用は無料**

## セットアップ手順

### Step 1：Tailscale をインストール

自宅の Mac とスマホ（iPhone / Android）の両方に Tailscale をインストールします。

- Mac: [tailscale.com](https://tailscale.com) からダウンロード
- iPhone / Android: App Store / Google Play で「Tailscale」を検索

### Step 2：同じアカウントでログイン

両方のデバイスで同じ Tailscale アカウント（Google / GitHub ログイン可）にサインインします。

### Step 3：Mac の Tailscale IP を確認

Tailscale メニューバーアイコンをクリックすると、Mac に割り当てられた IP が確認できます。

```
例: 100.xxx.xxx.xxx
```

**この IP アドレスをメモしておいてください**。以降の設定で使います。

### Step 4：Expo を Tailscale IP でバインドして起動

通常の `npx expo start` は、LAN の IP アドレスで Expo サーバーを起動します。Tailscale IP に合わせるために、環境変数 `REACT_NATIVE_PACKAGER_HOSTNAME` を指定します。

```bash
REACT_NATIVE_PACKAGER_HOSTNAME=100.xxx.xxx.xxx npx expo start
```

これで、Tailscale でつながっているスマホからは以下の URL で Expo Go を開けます。

```
exp://100.xxx.xxx.xxx:8081
```

## Node.js（Bot）から Expo を起動する方法

本書の Bot では、Slack のコマンド `run` を受け取ったときに Bot 側から Expo を自動起動します。Node.js の `spawn` を使って、環境変数を差し込んで起動するのがポイントです。

```typescript
import { spawn, ChildProcess } from "child_process";

let expoProcess: ChildProcess | null = null;
const TAILSCALE_IP = process.env.TAILSCALE_IP!; // .env に記述

// Expo 起動
expoProcess = spawn("npx", ["expo", "start"], {
  cwd: "/path/to/your-project",
  env: {
    ...process.env,
    REACT_NATIVE_PACKAGER_HOSTNAME: TAILSCALE_IP,
  },
  shell: true,
});
```

`cwd` でプロジェクトのディレクトリを指定し、`env` に Tailscale IP を注入することで、Expo が正しい IP でバインドされた状態で起動します。

## .env の設定

`TAILSCALE_IP` は `.env` に記述しておきます。

```
TAILSCALE_IP=100.xxx.xxx.xxx
```

こうしておくと、IP が変わったときも `.env` だけ更新すれば対応できます（Tailscale のIPはデバイスごとに固定されることが多いですが念のため）。

## 動作確認

外出先でスマホの Tailscale をオンにした状態で、Expo Go を開き以下を入力します。

```
exp://100.xxx.xxx.xxx:8081
```

アプリが起動すれば設定成功です。Mac 側で Gemini がコードを書き換えると、数秒でスマホのアプリ画面に反映されます。

次の章では、Slack Bot 実装で必ずハマる「2 つの大きな罠」と、その解決策を解説します。
