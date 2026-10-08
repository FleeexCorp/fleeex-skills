---
name: fleeex-api-keys
description: Create a fleeex app and its live and sandbox API keys through the fleeex MCP, and store them safely in the project's gitignored env file. Also covers key rotation, revocation and the ACTIVE_KEY_LIMIT_REACHED error. Use when the user says "create a fleeex app", "get a fleeex API key", "create a sandbox/test key", "rotate my fleeex key", "revoke a fleeex key", or when an integration needs FLEEEX_API_KEY.
---

# fleeex apps and API keys

Requires the fleeex MCP (see **fleeex-connect-mcp**). Every tool acts as the signed-in
account owner only.

## Keys at a glance

| Prefix | Mode | Effect |
|---|---|---|
| `flx_…` | `live` | Real models, real money from the end user's wallet. |
| `flx_test_…` | `sandbox` | No model called, fake balance, never billed, never in usage or webhooks. |

A key is **shown once**, in the result of the tool that minted it. fleeex cannot show it
again: lose it and you issue a new one.

## 1. Find or create the app

1. `list_apps`. If an app with the intended name exists, reuse its `appId` and skip to step 2.
2. Otherwise `create_app`:
   - `name` (1 to 100 chars): the name end users approve on the fleeex consent screen.
     Use one they will recognize.
   - `redirectUris` (optional, up to 10 absolute http(s) URLs, exact match): where fleeex
     may send users back after they connect their wallet. The MCP can only set them here;
     later changes go through the dashboard.
3. The result holds `appId`, `keyId` and `apiKey`: the first **live** key. Store it now
   (step 3).

## 2. Sandbox key for development

`create_api_key` with `{ appId, mode: "sandbox" }`. Returns `apiKey` (`flx_test_…`), shown
once. Store it now.

## 3. Store keys

Write values **directly into the env file**. Never echo a key in chat, in a log, in a
commit, or in a file that is not gitignored.

1. Pick the file the project already loads for local secrets: `.env.local` (Next.js, Vite,
   Nuxt, SvelteKit), otherwise `.env`. Follow the project's convention if it has another
   one (e.g. `.dev.vars` for Cloudflare Workers).
2. Check it is ignored: `git check-ignore -q <file>`. If not, add it to `.gitignore` first.
3. Write or replace these lines:

   ```bash
   # fleeex: sandbox key for local development (flx_test_…)
   FLEEEX_API_KEY=flx_test_...
   # fleeex: live key. Move it to the production secret store as FLEEEX_API_KEY.
   FLEEEX_LIVE_API_KEY=flx_...
   ```

4. Add the same names with empty values to `.env.example` (or the project's equivalent) so
   the next developer knows they exist.
5. Never use a public prefix (`NEXT_PUBLIC_`, `VITE_`, `EXPO_PUBLIC_`, `PUBLIC_`,
   `REACT_APP_`): those end up in the client bundle.

When reporting, name the variables and the `keyId`s, never the values.

## Rotate a key (no downtime)

1. `create_api_key` with the same `mode`: the old key keeps working.
2. Store the new key, deploy it, confirm traffic works.
3. `list_api_keys` to find the old `keyId`, then `revoke_api_key`. **Ask the user first**:
   revocation is immediate and cannot be undone.

## Revoke a key

`revoke_api_key({ appId, keyId })`. `keyId` comes from `list_api_keys`, it is not the key
itself. Destructive: get explicit approval, and make sure nothing deployed still uses it.

## Errors

| Code | Meaning | Fix |
|---|---|---|
| `ACTIVE_KEY_LIMIT_REACHED` | The app holds 20 active keys. | `list_api_keys`, ask which unused ones to revoke. |
| `APP_SUSPENDED` | The app is suspended by fleeex. | Contact `support@fleeex.dev`. |
| `NOT_FOUND` / `FORBIDDEN` | Unknown `appId` / app owned by another account. | Re-read `list_apps`. |
| `VALIDATION_FAILED` | An argument broke a rule (message lists which). | Fix the argument. |
| `UNAUTHORIZED` | The MCP session is gone. | Re-authenticate (**fleeex-connect-mcp**). |

## Usage reads

`get_app_usage({ appId, period?: "yyyy-mm" })` returns the month's totals, active users,
a daily series and top users by spend. Amounts are EUR micro-units (1 EUR = 1,000,000).
