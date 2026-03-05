---
title: "08 テストと Dry-run"
---

テスト手順と Canary / dry-run の運用ルールをまとめます。

内容:

- ローカルでのユニットテスト、統合テストの実行方法
- Canary ワークフローでの dry_run フラグの使い方
- テストアカウントの準備（X アカウント等）
- テストケース例: API の例外処理、rate limit ハンドリング

安全に本番へ移行するためのテスト戦略を示します。ローカルユニットテスト、統合テスト、ドライラン（Canary）の実行方法と CI 上での自動化を解説します。

テスト戦略の階層

- ユニットテスト: 各サービス（generator, storage, posting, utils）をモックして検証
- 統合テスト: Gemini や投稿 API 呼び出しを HTTP モック（nock 等）で疑似化して処理フローを検証
- E2E / Dry-run: 実際の投稿を行わない `DRY_RUN=true` モードで、本番ワークフローと同じスクリプトを走らせる

ローカル実行コマンド

```bash
# ユニットテスト
npm test

# カバレッジ
npm run test:coverage

# Dry-run 実行（生成のみ、投稿はしない）
DRY_RUN=true npm run generate-and-post -- --topic="テスト"
```

ユニットテストの実装例（Jest）

`__tests__/storage.spec.ts` や `__tests__/dry-run.spec.ts` が参考になります。

生成サービスのテスト（概念）:

```ts
import { mock } from 'jest-mock';
import { generateWithGemini } from '../services/generator.service';

jest.mock('node-fetch', () => /* モック実装 */);

test('generator returns trimmed text', async () => {
	const out = await generateWithGemini('入力');
	expect(out).toMatch(/^[^\s].*$/);
});
```

HTTP モックによる統合テスト

- `nock` を使って Gemini / Bluesky のエンドポイントをスタブ化し、異常系（429, 5xx）や成功系の挙動を検証する

Dry-run（Canary）運用

- Canary は `workflow_dispatch` で手動実行できるようにして、`dry_run` 入力をデフォルト `true` にするのが安全
- Dry-run では投稿呼び出しを行わず、投稿リクエストをログ/アーティファクトとして保存してレビューできるようにする

CI 組み込み例

- PR ごとにユニットテストと静的解析（ESLint, typecheck）を走らせる
- main ブランチには定期的に Dry-run（低頻度 cron）のワークフローを走らせる

検証ポイント（テスト項目）

- 生成されたテキストが字数制限内か
- 禁止ワードが含まれていないか
- 重複検出が正しく働くか（同一ハッシュの検出）
- API が異常を返したときの再試行とフォールバック

次の章では、運用でよくある問題とその切り分け手順をまとめます。
