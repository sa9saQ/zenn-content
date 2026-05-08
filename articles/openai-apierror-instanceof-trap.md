---
title: "APIUserAbortError extends APIError を知らずに CI を 1 件落とした話"
emoji: "🪤"
type: "tech"
topics: ["typescript", "openai", "llm", "claudecode"]
published: true
published_at: 2026-05-09 21:00
---

## TL;DR

- OpenAI SDK の `APIUserAbortError` は `APIError` の **サブクラス**
- `if (err instanceof APIError) {} else if (err instanceof APIUserAbortError) {}` の順で書くと、abort 系のエラーが APIError 側に先に捕まり、後続の分岐に到達しない
- `instanceof` チェックは「サブクラス → 親クラス」の順に書く。当たり前なんだが、外部 SDK の継承構造を知らないと踏む

ローカルで全部 pass、push、CI で 1 件落ちた。「TIMEOUT 期待で TRANSIENT が来た」。これだけ見て原因が思いつくなら読まなくていい。

---

## 何を作っていたか

AI でゲームを生成するサービスをやっていて、その中の「prompt-rewriter」というコンポーネントを書いていた。ユーザーが書いた自然言語のゲーム指示 (例: 「もっと難しくして、敵増やして」) を、Sonnet 4.6 で **provider 別に最適化された structured spec** に書き換える内部処理。Anthropic 向けには XML + Chain-of-Thought、OpenAI 向けには outcome-oriented な短文、と 1 回の呼び出しで 2 variants を同時生成する。

外向きに表示しないバックエンド処理なので、堅牢性が全て。`per-call timeout 5s` を設定しておいて、Sonnet が遅延したり落ちたりしたときは **raw prompt にフォールバック** する設計にした。

エラー分類は 4 段階:

- parent abort signal (ユーザーがキャンセル) → throw
- per-call 5s timeout → raw fallback (warning: `rewrite-timeout`)
- 5xx / 429 (transient) → raw fallback (warning: `rewrite-transient`)
- 401 / 403 / その他 4xx → throw

ここまでは普通。テストも書いた。

## 何が起きたか

ローカルで `pnpm typecheck` `lint` `format` `build` 全部 pass。CI に push。

戻ってきた 1 件の fail がこれ。

```
FAIL src/core/llm/prompt-rewriter.test.ts > 
  rewritePrompt — failure classification > 
  per-call timeout (parent abort なし、AbortError) → raw fallback (transient)

AssertionError: expected 'rewrite-transient' to be 'rewrite-timeout'

Expected: "rewrite-timeout"
Received: "rewrite-transient"
```

「per-call timeout」のテストで、warning が `rewrite-timeout` ではなく `rewrite-transient` になっている。**TIMEOUT 分岐に到達していない**。

該当テストはこう書いていた。

```ts
it("per-call timeout (parent abort なし、AbortError) → raw fallback (transient)", async () => {
  // parent signal は aborted ではない、独立の AbortError を APIUserAbortError として throw
  mockCreate.mockRejectedValueOnce(new APIUserAbortError());
  const { repo, inserted } = makeFakeRepo();

  const result = await rewritePrompt(
    { rawPrompt: "x", tier: "pro" },
    { apiKey: "k", repo },
  );

  expect(result.rewritten).toBe(false);
  expect(result.warning).toBe(REWRITE_WARNING.TIMEOUT);  // ← ここで fail
  expect(inserted[0]?.status).toBe("timeout");
});
```

`APIUserAbortError` を投げているのに、TIMEOUT じゃなくて TRANSIENT に来る。なんで?

## 原因: SDK の継承構造を見ていなかった

catch ブロックはこう書いていた。`APIError` を先にチェックしていた。

```ts
} catch (err) {
  // parent abort は最優先で throw
  if (options.signal?.aborted) {
    throw err instanceof Error ? err : new Error(String(err));
  }

  // 4xx / 401 / 403 → throw / 5xx / 429 → raw fallback (TRANSIENT)
  if (err instanceof APIError) {        // ← ここで先に捕まっている
    const status = err.status ?? 0;
    if (status === 401 || status === 403) throw err;
    if (status >= 400 && status < 500 && status !== 408 && status !== 429) throw err;
    // 5xx / 408 / 429 → raw fallback
    return rawFallback(rawPrompt, REWRITE_WARNING.TRANSIENT);
  }

  // per-call 5s timeout → raw fallback (TIMEOUT)
  if (err instanceof APIUserAbortError || ...) {  // ← ここに到達しない
    return rawFallback(rawPrompt, REWRITE_WARNING.TIMEOUT);
  }
  ...
}
```

「APIError を最初に書いて、abort はその後」。ぱっと見、自然な順序に見える。

罠はここ。OpenAI SDK の error class の継承構造はこうなっている。

| Error class                    | 継承元                  | いつ起きる              |
| ------------------------------ | ----------------------- | ----------------------- |
| `APIError`                     | `Error`                 | base class              |
| `APIUserAbortError`            | `APIError`              | request abort / timeout |
| `APIConnectionError`           | `APIError`              | network                 |
| `APIConnectionTimeoutError`    | `APIConnectionError`    | network timeout         |
| `RateLimitError`               | `APIError`              | 429                     |
| `AuthenticationError`          | `APIError`              | 401                     |
| `BadRequestError`              | `APIError`              | 400                     |

**全部 `APIError` のサブクラス**。`APIUserAbortError` も例外じゃない。

つまり `if (err instanceof APIError)` は abort error も network error も rate limit error も全部捕まえる。先に書いた分岐が後続を全部食う。

`status` プロパティで分岐していたから、abort error (status undefined → 0) は「5xx でも 429 でもない」 → 「その他 4xx」判定にも引っかからず → 最後の `return rawFallback(..., TRANSIENT)` に流れていた。

```ts
const status = err.status ?? 0;
if (status === 401 || status === 403) throw err;
if (status >= 400 && status < 500 && status !== 408 && status !== 429) throw err;
// status === 0 はここを通過
// 5xx / 408 / 429 → raw fallback (TRANSIENT)  ← abort も巻き込まれる
return rawFallback(rawPrompt, REWRITE_WARNING.TRANSIENT);
```

**status が 0 のままでも TRANSIENT 判定**。つまり API call が一度も走ってない abort 状態が「サーバーが 5xx 返した」と同じ扱いになっていた。やばい。

ローカルの typecheck / lint は通る。型は正しい。意味だけ違う。CI test までやらないと永遠に気づかない種類のバグ。

## 修正

`APIUserAbortError` チェックを `APIError` の **前** に置く。これだけ。

```ts
} catch (err) {
  const latencyMs = Date.now() - start;

  // parent abort は最優先で throw (per-call timeout / 4xx と区別)
  if (options.signal?.aborted) {
    throw err instanceof Error ? err : new Error(String(err));
  }

  // per-call 5s timeout (parent abort は上で除外済) → raw fallback
  // 注: OpenAI SDK では `APIUserAbortError extends APIError` のため、
  // この判定を APIError ブランチより **先に** 行わないと TIMEOUT 分岐に
  // 到達せず TRANSIENT に誤判定する (CI fail で発覚、修正)。
  if (
    err instanceof APIUserAbortError ||
    (err instanceof Error && err.name === "AbortError")
  ) {
    return rawFallback(rawPrompt, REWRITE_WARNING.TIMEOUT);
  }

  // 4xx / 401 / 403 → throw / 5xx / 429 → raw fallback
  if (err instanceof APIError) {
    const status = err.status ?? 0;
    if (status === 401 || status === 403) throw err;
    if (status >= 400 && status < 500 && status !== 408 && status !== 429) throw err;
    return rawFallback(rawPrompt, REWRITE_WARNING.TRANSIENT);
  }

  // network / 未知 → raw fallback (TRANSIENT)
  return rawFallback(rawPrompt, REWRITE_WARNING.TRANSIENT);
}
```

順序を **「parent abort signal → APIUserAbortError → APIError → 未知 Error」** の 4 段階に統一。継承の深い順から書く。

CI が緑になった。1 行も削らず、2 ブロックを入れ替えただけ。

## 教訓

`instanceof` チェックは **サブクラス → 親クラス** の順。これが基本。

JavaScript / TypeScript で書いていると、自分が定義した class なら継承構造を覚えているから順序を間違えない。問題は **外部 SDK の class**。OpenAI SDK の `APIUserAbortError` を見ても、名前から「abort 用の専用 class」だと思い込む。「`APIError` のサブクラスである」という事実は、SDK の source を覗かないとわからない。

防御策をいくつか。

**1. SDK の error class 構造を表にして仕様書に貼る**

私は ARCHITECTURE.md に、上の表をそのまま転記した。次に書く人 (未来の自分) は、catch ブロックを書く前にその表を見れば順序を間違えない。

**2. テストで実 SDK の error class を投げる**

mock で素の `Error` を投げると、継承関係の差分が出ない。テストは必ず実 SDK の error をそのまま使う。

```ts
// ❌ 継承構造を再現できないので罠を発見できない
mockCreate.mockRejectedValueOnce(new Error("aborted"));

// ✅ 実 SDK の class、`instanceof` 判定が正しく動く
mockCreate.mockRejectedValueOnce(new APIUserAbortError());
```

私の場合これは合っていた。が、catch 順序が間違っていたから fail した。順序の方を直せばよかった、というオチ。

**3. ローカル typecheck だけで安心しない**

「`pnpm typecheck` `lint` `format` `build` 全部 green、push」と書いた瞬間に少し安心していた。型は通る。lint も通る。build も通る。でも CI test まで走らせないと **意味の正しさ** は検証されない。

これは Cloudflare Workers の sandbox 制約で `pnpm test` がローカルで走らないプロジェクトだったので、CI 任せで運用していた。普段ならローカルで test 流して気づけたはず。「ローカルで全部 pass」の射程をちゃんと意識する。

## 余談: 他の SDK でも同じ罠

OpenAI SDK 以外でも、HTTP client / SDK の error 階層は深い継承を持つことが多い。

- **Anthropic SDK**: `APIError` 配下に `BadRequestError` `AuthenticationError` `PermissionDeniedError` など、OpenAI と似た構造
- **AWS SDK v3**: `ServiceException` 配下に各種 service error
- **Stripe SDK**: `StripeError` 配下に `StripeCardError` `StripeRateLimitError` など

いずれも `instanceof` チェック順で同じ罠を踏む。SDK を採用したら **error class の継承図を最初に確認する** のが結局一番早い。

## 参考

- OpenAI SDK の error class: [openai/openai-node](https://github.com/openai/openai-node/blob/master/src/error.ts)
- TypeScript narrowing と `instanceof`: [TypeScript Handbook - Narrowing](https://www.typescriptlang.org/docs/handbook/2/narrowing.html)
- 私の実装した prompt-rewriter (Enova): `src/core/llm/prompt-rewriter.ts` の catch 順序が今回の核 (修正後 PR-15 で確定)

---

ローカルで全部 pass、push、CI で落ちる。30 分くらい原因分からなかった。「型は正しい、なのに動きが違う」って一番気持ち悪い種類のバグ。

`APIUserAbortError extends APIError`、たった 1 行の継承宣言。これを見落としただけで、abort error が timeout error になり、サーバーエラーと同じ raw fallback に流される。

書いてみると当たり前すぎる教訓だが、当たり前を踏むのが一番悔しい。
