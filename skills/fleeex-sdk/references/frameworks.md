# fleeex with JS/TS frameworks

All recipes target `https://api.fleeex.dev/v1` (note `/v1`), send the app key as bearer and
the end user id as `x-fleeex-user`. The user id always comes from the server-side session.

## Next.js App Router route handler

```ts
// app/api/chat/route.ts
import { PaymentRequiredError } from '@fleeex/sdk';
import { fleeexFor } from '@/lib/fleeex';
import { getSession } from '@/lib/auth'; // the project's own session helper

const HTTP_PAYMENT_REQUIRED = 402;

export async function POST(req: Request): Promise<Response> {
  const session = await getSession();
  if (!session) {
    return new Response('Unauthorized', { status: 401 });
  }

  const { messages } = await req.json();

  try {
    const completion = await fleeexFor(session.userId).chat.completions.create({
      model: 'nova-lite',
      messages,
    });
    return Response.json(completion.choices[0]?.message);
  } catch (err) {
    if (!(err instanceof PaymentRequiredError)) {
      throw err;
    }
    return Response.json(
      { error: err.code ?? 'PAYMENT_REQUIRED', topupUrl: err.topupUrl ?? null },
      { status: HTTP_PAYMENT_REQUIRED },
    );
  }
}
```

Validate `messages` with the project's validation library before forwarding.

## Express

```ts
app.post('/api/chat', requireAuth, async (req, res, next) => {
  try {
    const completion = await fleeexFor(req.user.id).chat.completions.create({
      model: 'nova-lite',
      messages: req.body.messages,
    });
    res.json(completion.choices[0]?.message);
  } catch (err) {
    if (err instanceof PaymentRequiredError) {
      res.status(402).json({ error: err.code ?? 'PAYMENT_REQUIRED', topupUrl: err.topupUrl ?? null });
      return;
    }
    next(err);
  }
});
```

## NestJS

Provide the factory as an injectable service (singleton; the user id is a method argument,
never a field), and map `PaymentRequiredError` in an exception filter to the same `402`
body.

## Vercel AI SDK

Use the OpenAI-compatible provider: it speaks chat completions, which is what fleeex
serves. (`@ai-sdk/openai` defaults to the Responses API on recent versions, which fleeex
does not serve.)

```bash
npm install @ai-sdk/openai-compatible
```

```ts
import { createOpenAICompatible } from '@ai-sdk/openai-compatible';

export const fleeex = createOpenAICompatible({
  name: 'fleeex',
  baseURL: 'https://api.fleeex.dev/v1',
  apiKey: process.env.FLEEEX_API_KEY,
  includeUsage: true,
  // generateObject must send json_schema: fleeex rejects json_object.
  supportsStructuredOutputs: true,
});
```

Pass the user per call:

```ts
const result = streamText({
  model: fleeex('nova-lite'),
  messages,
  headers: { 'x-fleeex-user': session.userId },
});
```

`402` arrives as `APICallError` with `statusCode === 402`; the top-up link is in the raw
body:

```ts
import { APICallError } from 'ai';

if (APICallError.isInstance(err) && err.statusCode === 402) {
  const body = JSON.parse(err.responseBody ?? '{}');
  // body.topupUrl (may be absent), body.error?.code
}
```

Do not set `seed`, `frequencyPenalty` or `presencePenalty` (non-zero) in call settings: they
are rejected. Use `@fleeex/sdk` alongside for `getConnection()`, `getBalance()`,
`getUsageSummary()` and `verifyWebhook()`.

## LangChain JS

```ts
import { ChatOpenAI } from '@langchain/openai';

const model = new ChatOpenAI({
  model: 'nova-lite',
  apiKey: process.env.FLEEEX_API_KEY,
  configuration: {
    baseURL: 'https://api.fleeex.dev/v1',
    defaultHeaders: { 'x-fleeex-user': userId },
  },
});
```

Build one per request (the header is per user). Use `withStructuredOutput(schema, { method:
'jsonSchema' })`, never `jsonMode`. Verify with a sandbox call: LangChain versions differ in
which defaults they send, and anything rejected comes back as a `400` naming it.

## Raw OpenAI SDK (no wrapper)

```ts
const client = new OpenAI({
  apiKey: process.env.FLEEEX_API_KEY,
  baseURL: 'https://api.fleeex.dev/v1',
  defaultHeaders: { 'x-fleeex-user': userId },
});
```

Byte-identical on the wire to `@fleeex/sdk`, but a `402` surfaces as `APIError` with only
the `error` field, so `topupUrl` is lost: call `getConnection()` (from `@fleeex/sdk`, or
`POST /connect`, see [python.md](python.md)) to get a link.
