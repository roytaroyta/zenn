---
title: "05 投稿済み管理と重複検出"
---

投稿済み URL の管理と重複検出の設計についてまとめます。

目的:

- 同じコンテンツを複数回投稿しない
- 失敗時の再試行で重複を避ける

現状（要確認）:

- `data/posted.json` などに投稿履歴がある想定

検討すべき案:

1. 単純な URL のハッシュ保存（軽量）
2. 永続ストレージ（SQLite / DynamoDB / Cloud Firestore）
3. TTL を持たせたストレージ（短期重複回避）

実装例（擬似コード）:

- 投稿前にハッシュで存在確認
- 投稿後に成功を確定してから履歴へ追記

注意点:

- 同時実行時の競合対策（ロック、楽観的ロック）
- 履歴ファイルの肥大化対策

自動投稿システムでの重複検出は非常に重要です。ここでは小規模なローカル実装から、運用レベルのストレージ設計まで具体的に説明します。

データモデル（推奨）

投稿履歴エントリの例:

```json
{
  "id": "bsky-post-12345",
  "hash": "sha256-...",
  "text_preview": "生成した本文の先頭部分...",
  "source": "generator/topic-x",
  "postedAt": "2026-03-06T09:00:00Z",
  "meta": { "tokens": 123 }
}
```

重複判定の基本ルール

- コンテンツハッシュ（本文を正規化して SHA-256）を一意キーとして判定するのが最も確実
- 必要に応じて `source`（プロンプトやトピック）やメディアIDも合わせて判定する

軽量（ファイルベース）実装（TypeScript）

`storage.service.ts` の簡易実装例:

```ts
import fs from "fs/promises";
import crypto from "crypto";

const POSTED_PATH = process.env.POST_STORAGE_PATH ?? "./data/posted.json";

type Posted = {
  id: string;
  hash: string;
  text_preview?: string;
  postedAt: string;
};

async function readPosted(): Promise<Posted[]> {
  try {
    const raw = await fs.readFile(POSTED_PATH, "utf-8");
    return JSON.parse(raw) as Posted[];
  } catch (e) {
    return [];
  }
}

function hashText(text: string) {
  return crypto.createHash("sha256").update(text).digest("hex");
}

export async function isDuplicate(text: string) {
  const h = hashText(normalize(text));
  const list = await readPosted();
  return list.some((item) => item.hash === h);
}

export async function savePosted(entry: Omit<Posted, "postedAt">) {
  const list = await readPosted();
  list.push({ ...entry, postedAt: new Date().toISOString() });
  // 原子的に書き込む
  await fs.writeFile(POSTED_PATH, JSON.stringify(list, null, 2), "utf-8");
}

function normalize(s: string) {
  return s.replace(/\s+/g, " ").trim();
}
```

注意: 上記は単純化した実装で、複数のワーカーが同時に実行される環境では競合が発生する可能性があります。対策は以下のとおりです:

- ファイルロック/ロックファイルを使う（`proper-lockfile` 等のライブラリ）
- 外部 DB（SQLite/Postgres/DynamoDB）で一意インデックスを使う
- 保存前に二段階認証的に重複を再チェックする（楽観的ロック）

運用レベルの設計案

- 小規模: ファイルベースで十分だが、同時実行が増えると脆弱
- 中〜大規模: RDS / Postgres や DynamoDB を使い、`hash` に UNIQUE 制約を付ける
- TTL を設定したい場合: Redis や DynamoDB TTL 属性を使う

例: DynamoDB を使う場合のメリット

- 同時実行でも一意性制約を DB 側で保証できる（条件付き書き込み）
- スケーラブルで運用の負担が少ない

履歴の肥大化対策

- 古いエントリを一定期間でアーカイブまたは削除（例: 180 日）
- 重要メタデータのみを残し、全文は別の長期保存先へ移す

テスト方針

- `isDuplicate` と `savePosted` を単体テストでカバーする
- 並列実行の疑似環境で動作確認を行う（integration テスト）

次の章では、生成と投稿の可観測性を高めるためのモニタリングとログ設計を解説します。
