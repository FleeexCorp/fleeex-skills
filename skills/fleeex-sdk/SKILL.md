---
name: fleeex-sdk
description: Wire @fleeex/sdk (or the raw OpenAI SDK, Vercel AI SDK, LangChain, Python openai) into a project's backend so its LLM calls go through fleeex and are billed to each end user's wallet. Covers install, the per-user client, model choice, the supported parameter surface, 402 top-up handling, the connect-wallet gate, streaming and usage display. Use when the user says "use the fleeex SDK", "migrate my OpenAI calls to fleeex", "bill AI calls to the user's wallet", "handle fleeex 402", "add a connect fleeex button", or after keys exist and the code is not wired yet.
---

# Integrate the fleeex SDK

The wire is **iso-OpenAI**: `POST https://api.fleeex.dev/v1/chat/completions` with
`Authorization: Bearer <FLEEEX_API_KEY>` and `x-fleeex-user: <userId>`. `@fleeex/sdk` is a
thin wrapper over the OpenAI SDK that sets both, and adds billing helpers and webhook
verification. Docs: https://docs.fleeex.dev.

Prerequisite: `FLEEEX_API_KEY` exists in the env file (**fleeex-api-keys**).

## 1. Map the project

- Backend runtime and framework, package manager (lockfile), TS or JS, ESM or CJS.
- Existing LLM calls: `grep -rnE "from ['\"]openai['\"]|@ai-sdk/|langchain|chat\.completions|responses\.create" --exclude-dir=node_modules`.
- Where the server knows the signed-in user (session helper, auth middleware).

Pick the path:

| Project | Path |
|---|---|
| TS/JS backend, plain OpenAI SDK or no LLM yet | `@fleeex/sdk` (below) |
| Vercel AI SDK, LangChain JS | [references/frameworks.md](references/frameworks.md) |
| Python, or any other language | [references/python.md](references/python.md) (raw OpenAI client + HTTP) |

## 2. Install

```bash
npm install @fleeex/sdk openai
```

Use the project's package manager. `openai` (>= 4) is a peer dependency. Node 18+.

## 3. One fleeex module

Create a single server-only module (e.g. `src/lib/fleeex.ts`, next to existing server
utilities) and route every fleeex call through it:

```ts
import { FleeexClient } from '@fleeex/sdk';

const apiKey = process.env.FLEEEX_API_KEY;

/** fleeex client billing the given end user's wallet. Server-side only. */
export function fleeexFor(userId: string): FleeexClient {
  if (!apiKey) {
    throw new Error('FLEEEX_API_KEY is not set');
  }

  return new FleeexClient({ apiKey, userId });
}
```

- `userId` is the **stable id of the signed-in user** from the server-side session. Never
  read it from the request body, query or a client-set header: whoever controls it chooses
  whose wallet pays. Prefer an internal id over an email (it shows up in webhook payloads
  and usage reports).
- `baseURL` defaults to `https://api.fleeex.dev`; the `FLEEEX_BASE_URL` env var overrides
  it without code changes.
- In Next.js add `import 'server-only';` at the top if the project uses that package.

## 4. Route the calls

`client.chat.completions.create` **is** the OpenAI method with OpenAI's types, so existing
calls keep their shape:

```ts
const completion = await fleeexFor(session.userId).chat.completions.create({
  model: 'nova-lite',
  messages: [{ role: 'user', content: prompt }],
});
```

Check each call against the parameter surface: unsupported parameters are a `400` naming
the parameter, never silently dropped. Common fixes: drop `seed`, use `n: 1`, replace
`response_format: { type: 'json_object' }` with `json_schema`, inline images as
`data:image/...;base64,` URIs. Full list: [references/parameters.md](references/parameters.md).

Only chat completions and model listing go through fleeex. `responses.create` must be
rewritten as chat completions. Embeddings, images, audio and other endpoints are not
proxied: leave them on their current provider and say so in the report.

## 5. Choose the model

Do not hardcode a list or map OpenAI model names by guess. Ask the catalog (scoped to the
app; everything returned is callable):

```ts
import OpenAI from 'openai';

const catalog = new OpenAI({ apiKey: process.env.FLEEEX_API_KEY!, baseURL: 'https://api.fleeex.dev/v1' });
for await (const m of await catalog.models.list()) console.log(m.id, m.owned_by);
```

Use **aliases** (`nova-lite`), not versioned provider ids: fleeex re-points aliases. If the
user cares about a quality or cost tier, show them the list and let them pick. A model
reserved to other apps answers `403 MODEL_NOT_ENTITLED`.

## 6. Handle 402 (no funded wallet)

Any proxied call can throw `PaymentRequiredError`. Handle it in **one** place and turn it
into something the frontend can act on:

```ts
import { PaymentRequiredError } from '@fleeex/sdk';

try {
  // ... fleeex call
} catch (err) {
  if (!(err instanceof PaymentRequiredError)) {
    throw err;
  }

  // topupUrl is single-use and short-lived: pass it as-is, never rewrite it.
  return Response.json({ error: err.code ?? 'PAYMENT_REQUIRED', topupUrl: err.topupUrl ?? null }, { status: 402 });
}
```

Frontend: on `402` with a `topupUrl`, redirect the user there (or open it); without one,
show a message:

| `err.code` | `topupUrl` | Show |
|---|---|---|
| `PAYMENT_REQUIRED` | yes on a proxied call | "Fund your fleeex wallet" then redirect. |
| `APP_SPEND_CAP_EXCEEDED` | usually yes, sometimes none | The user's monthly cap for this app is reached. |
| `WALLET_SUSPENDED` | never | Wallet suspended: contact `support@fleeex.dev`. |

`getBalance()` also throws `PaymentRequiredError` when the user has no wallet connected to
the app, but **without** `topupUrl`: take the link from `getConnection().connectUrl` (step 7).

Alternative: the `onPaymentRequired` hook in the `FleeexClient` options fires on every
mapped `402` before the throw.

## 7. Connect gate (recommended)

Check the wallet before the first AI call instead of waiting for a `402`. It never charges
and never throws `402`:

```ts
const { connected, funded, connectUrl } = await fleeexFor(userId).getConnection({
  redirectUri: 'https://app.example.com/fleeex/return', // must be registered on the app
});

if (!connected || !funded) {
  // show "Connect fleeex" -> connectUrl (sign in, approve this app, top up, come back)
}
```

`redirectUri` must exactly match one of the app's registered redirect URIs, otherwise
`FleeexApiError` 400. Omit it if none are registered. Add a return route that simply
re-checks `getConnection()` and resumes.

## 8. Streaming and usage

- `stream: true` works as in OpenAI. Add `stream_options: { include_usage: true }` to get
  the billed usage in the last chunk.
- A stream that fails mid-way throws `APIError` with `code` `UPSTREAM_PROVIDER_ERROR`,
  `UPSTREAM_THROTTLED` or `UPSTREAM_STREAM_TRUNCATED`. What streamed before is billed.
- `getBalance()` returns `{ balanceMicros, currency: 'EUR' }`; `getUsageSummary()` returns
  month-to-date `spentMicros`, `totalTokens`, `requestCount`. 1 EUR = 1,000,000 micros.
  Format with `(micros / 1e6).toFixed(2)`.

## 9. Verify

1. Typecheck and run the tests.
2. One call with the sandbox key (`flx_test_…`): expect a canned answer and the
   `x-fleeex-mode: sandbox` response header. See **fleeex-sandbox** to force `402` states.
3. Grep the client bundle sources for `FLEEEX_`: nothing may reference it outside server code.

## Errors

| Error | When |
|---|---|
| `PaymentRequiredError` | `402`: `topupUrl?`, `code?`, `correlationId?` |
| `FleeexApiError` | Other non-2xx from a billing helper: `status`, `code?`, `correlationId?` |
| `OpenAI.APIError` | Proxy errors other than `402`: `400` bad parameter or model, `403 MODEL_NOT_ENTITLED`, `429` throttled, `502` provider failure |

Log `correlationId` when present: it is what fleeex support needs.
