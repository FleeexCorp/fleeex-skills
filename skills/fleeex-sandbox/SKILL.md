---
name: fleeex-sandbox
description: Test a fleeex integration for free with a sandbox key (flx_test_), check the canned response, and force the hard paths (insufficient funds 402, spend cap, suspended wallet) to verify the app's top-up and error handling. Use when the user says "test fleeex without paying", "fleeex sandbox", "simulate a 402", "test the top-up flow", "test insufficient funds", or after wiring the SDK.
---

# fleeex sandbox

A sandbox key (`flx_test_…`) runs the full fleeex pipeline (same wire, validation, model
entitlements, rate limits, `402` shapes) but **calls no model** and spends a **fake balance**,
funded automatically on a test user's first call. Nothing is billed, nothing shows in usage
metrics, invoices or webhooks.

Mint one with the MCP: `create_api_key({ appId, mode: "sandbox" })` (see **fleeex-api-keys**).

## 1. Smoke test

Run one call through the project's own fleeex module with `FLEEEX_API_KEY` set to the
sandbox key and a test user id (e.g. `test-user-1`). Or directly:

```bash
curl -sS -D - https://api.fleeex.dev/v1/chat/completions \
  -H "authorization: Bearer $FLEEEX_API_KEY" \
  -H "x-fleeex-user: test-user-1" \
  -H "content-type: application/json" \
  -d '{"model":"nova-lite","messages":[{"role":"user","content":"ping"}]}'
```

Load the key from the env file into the shell (`set -a; . ./.env.local; set +a`) rather
than pasting it. Expect `200`, the header `x-fleeex-mode: sandbox` and a canned answer.
A `400` names a rejected parameter: fix it (see the `fleeex-sdk` skill's
`references/parameters.md`); it would fail the same way in live.

## 2. Force the hard paths

Sandbox state is set through the **control plane**, authenticated with the owner's fleeex
**session token** (not an API key, not the MCP). The MCP does not expose these calls, so the
user runs them, or exports `FLEEEX_SESSION_TOKEN` in their own shell for you to use. Never
ask for the token in chat.

```bash
BASE=https://api.fleeex.dev/apps/<appId>/sandbox
AUTH="authorization: Bearer $FLEEEX_SESSION_TOKEN"

# Insufficient funds: next call -> 402 PAYMENT_REQUIRED with a placeholder topupUrl
curl -X POST "$BASE/users/test-user-1/reset" -H "$AUTH" -H 'content-type: application/json' -d '{"balanceMicros":0}'

# Monthly spend cap: next call -> 402 APP_SPEND_CAP_EXCEEDED
curl -X POST "$BASE/users/test-user-1/reset" -H "$AUTH" -H 'content-type: application/json' -d '{"spendCapMicros":1}'

# Suspended wallet: next call -> 402 WALLET_SUSPENDED, no topupUrl
curl -X POST "$BASE/users/test-user-1/reset" -H "$AUTH" -H 'content-type: application/json' -d '{"suspended":true}'

# Back to the default grant
curl -X POST "$BASE/users/test-user-1/reset" -H "$AUTH"

# State of every test user (fake balance, reservations, requests, tokens)
curl "$BASE" -H "$AUTH"
```

Resets are throttled (10 per minute). A reset clears reservations and counters but not
idempotency markers: use a new `Idempotency-Key` after a reset if you send one.

For each state, check the app's behavior end to end: the backend returns `402` with the
right `code`, the frontend redirects when `topupUrl` exists and shows a message otherwise.
The sandbox `topupUrl` is a placeholder: it cannot take a payment.

## Limits

- `getBalance()` and `getUsageSummary()` read the **real** wallet even with a test key
  (a `402` and zeros). Read fake balances from `GET /apps/<appId>/sandbox`.
- No webhooks in sandbox: test the receiver with `signWebhook` (see **fleeex-webhooks**).
- A test key cannot be promoted to live. Switch `FLEEEX_API_KEY` to the live key in
  production only.
