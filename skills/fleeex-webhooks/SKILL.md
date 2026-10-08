---
name: fleeex-webhooks
description: Add a fleeex webhook receiver to a project (signature verification with verifyWebhook, dedup on event id, fast 2xx) and register the subscription through the fleeex MCP (set_webhook), storing the signing secret safely. Events are balance.low, topup.completed and connection.created. Use when the user says "add fleeex webhooks", "notify me when a user's balance is low", "react to a fleeex top-up", "register a webhook endpoint", "rotate the fleeex webhook secret".
---

# fleeex webhooks

One https endpoint per app. fleeex POSTs signed JSON events to it.

| Event | Fires when | `data` |
|---|---|---|
| `balance.low` | the user's spendable balance crosses below **your** threshold (once per crossing) | `userId`, `availableMicros`, `thresholdMicros`, `currency` |
| `topup.completed` | the user funded their wallet (verified payment) | `userId`, `balanceMicros`, `availableMicros`, `currency` |
| `connection.created` | a user connected their wallet to your app for the first time | `userId`, `connectedAt` |

`userId` is the id your app sent in `x-fleeex-user`. Amounts are EUR micros (1 EUR = 1,000,000).
Sandbox keys never produce webhooks.

## 1. Choose events

Ask what the app should do with each event; subscribe only to those. Typical:
`balance.low` (warn the user before they are cut off, threshold e.g. `2000000` = 2 EUR),
`topup.completed` (unblock a paused feature), `connection.created` (onboarding analytics).

## 2. Build the receiver

Path suggestion: `/api/webhooks/fleeex` (or the project's webhook convention).

Contract:

1. Read the **raw body** before any JSON parsing. Re-serialized JSON does not verify.
2. `verifyWebhook({ rawBody, headers, secret: process.env.FLEEEX_WEBHOOK_SECRET })` from
   `@fleeex/sdk` (Node runtime). On `WebhookSignatureError` answer `401` with no side effect.
3. **Deduplicate on `event.id`**: delivery is at-least-once and a redelivery carries the same
   id and body. Persist handled ids with a unique constraint in the project's database
   (insert first; a duplicate-key error means "already handled", answer `2xx`).
4. Answer `2xx` within **5 seconds**. Enqueue slow work (the project's queue or background
   job system) instead of doing it inline.
5. Exclude the route from auth middleware, CSRF protection and body parsers that consume
   the stream.

```ts
import { verifyWebhook, WebhookSignatureError, type WebhookEvent } from '@fleeex/sdk';

const HTTP_UNAUTHORIZED = 401;

export async function POST(req: Request): Promise<Response> {
  let event: WebhookEvent;
  try {
    event = verifyWebhook({
      rawBody: await req.text(),
      headers: req.headers,
      secret: process.env.FLEEEX_WEBHOOK_SECRET!,
    });
  } catch (err) {
    if (err instanceof WebhookSignatureError) {
      return Response.json({ error: err.reason }, { status: HTTP_UNAUTHORIZED });
    }
    throw err;
  }

  // Insert-first under a unique constraint: concurrent redeliveries cannot both pass.
  if (!(await recordEventId(event.id))) {
    return Response.json({ received: true });
  }

  switch (event.type) {
    case 'balance.low':
      await enqueueLowBalanceNotice(event.data.userId, event.data.availableMicros);
      break;
    case 'topup.completed':
      await enqueueTopupHandled(event.data.userId);
      break;
    case 'connection.created':
      break;
  }

  return Response.json({ received: true });
}
```

Raw body per framework:

| Framework | Raw body |
|---|---|
| Next.js App Router, Hono, Remix, SvelteKit, Workers (fetch `Request`) | `await req.text()` |
| Next.js Pages Router | `export const config = { api: { bodyParser: false } }`, read the stream |
| Express | `express.raw({ type: 'application/json' })` on this route, mounted **before** `express.json()`; `req.body` is a `Buffer` |
| NestJS | `NestFactory.create(AppModule, { rawBody: true })`, then `req.rawBody` |
| Fastify | `addContentTypeParser('application/json', { parseAs: 'string' }, ...)` scoped to the route |
| Python | see the `fleeex-sdk` skill, `references/python.md` |

`verifyWebhook` needs `node:crypto`. On an edge runtime, force the Node runtime for this
route (Next.js: `export const runtime = 'nodejs'`).

## 3. Test locally

Write a test that signs a body with `signWebhook` and posts it to the handler:

```ts
import { signWebhook, WEBHOOK_SIGNATURE_HEADER } from '@fleeex/sdk';

const secret = 'whsec_test';
const rawBody = JSON.stringify({
  id: 'evt_test_1',
  type: 'balance.low',
  createdAt: new Date().toISOString(),
  appId: 'app_test',
  data: { userId: 'u1', availableMicros: 1500000, thresholdMicros: 2000000, currency: 'EUR' },
});
const headers = { [WEBHOOK_SIGNATURE_HEADER]: signWebhook({ secret, rawBody }), 'content-type': 'application/json' };
```

Cover: valid event handled once, same id twice handled once, tampered body `401`.

## 4. Register (needs a public https URL)

fleeex refuses `localhost`, private addresses and plain http. Use the deployed URL, or a
tunnel during development (`cloudflared tunnel --url http://localhost:3000`,
`ngrok http 3000`). Ask the user which URL to register; do not start a tunnel unasked.

1. `get_webhook({ appId })`. `NOT_FOUND` means no subscription yet.
2. `set_webhook({ appId, endpointUrl, events, lowBalanceThresholdMicros })`.
   - The arguments are the **complete desired state**: an omitted field is cleared.
   - `lowBalanceThresholdMicros` (1 to 1,000,000,000) is required with `balance.low` and
     refused without it.
3. **Only when the call created the subscription**, the result holds `signingSecret`
   (`whsec_…`), shown once. Write it straight into the gitignored env file as
   `FLEEEX_WEBHOOK_SECRET` (same rules as **fleeex-api-keys**); never print it.
4. Subscription already existed and the secret is unknown locally? `rotate_webhook_secret`
   mints a new one, but the old one **stops verifying at once**: ask the user first and
   update every deployed receiver right after.

Remind the user to set `FLEEEX_WEBHOOK_SECRET` in production.

## Delivery rules to design for

- Headers: `x-fleeex-signature` (`t=<unix>,v1=<hex>`), `x-fleeex-event-id`,
  `x-fleeex-event-type`, `x-fleeex-delivery-attempt`.
- Signatures older or newer than 300 s are rejected by `verifyWebhook` (`expired`).
- Non-2xx, timeout (5 s) or connection error: redelivered on a fixed interval (no
  exponential backoff), up to six attempts in all (`x-fleeex-delivery-attempt` says which).
  After that the event is **not delivered again**. Redirects are not followed.
- fleeex never disables a subscription for failing deliveries. `status: "disabled"` only
  comes from the owner.
- Payloads never contain prompts, emails, amounts paid or fleeex-internal ids.
