---
name: fleeex-setup
description: End-to-end fleeex integration for the current project. Connects the fleeex MCP, creates (or reuses) the fleeex app, mints sandbox and live API keys into a gitignored env file, wires @fleeex/sdk (or the raw OpenAI SDK) into the backend, adds the connect/top-up flow, and optionally registers a webhook. Use when the user says "integrate fleeex", "add fleeex to my app", "set up fleeex", "bill AI usage to my users' wallets", "make my users pay their own AI usage", or /fleeex-setup.
---

# fleeex setup

fleeex is a prepaid AI wallet owned by the **end user** plus an OpenAI-compatible LLM
proxy (`https://api.fleeex.dev/v1`). Your app calls models through fleeex with its API key
and the id of the user it vouches for; the user's own wallet pays. One user, one wallet,
across every app that integrates fleeex.

This skill orchestrates the other fleeex skills. Run the steps in order, skip what already
exists, and never redo a step whose result is already in the project.

## 0. Preflight

1. Locate the **backend**: the code that runs on a server (API routes, server actions, a
   NestJS/Express/FastAPI service). The fleeex API key is a server-side secret: it never
   ships in a browser, mobile or desktop bundle. If the project is client-only, stop and
   tell the user a backend (even one serverless route) is required.
2. Detect the stack (lockfile, `package.json`, `pyproject.toml`, `requirements.txt`) and the
   existing LLM calls (`grep -rE "openai|@ai-sdk/|langchain|anthropic|bedrock"` outside
   `node_modules`).
3. Detect how the app identifies its signed-in user server-side (session, JWT, auth
   library). The fleeex `userId` comes from there.
4. Ask the user, in **one** message, only what you cannot infer:
   - the app **name** end users will see on the fleeex consent screen (default: the
     product name from `package.json`/README);
   - the URL fleeex sends users back to after connecting their wallet (optional, e.g.
     `https://app.example.com/fleeex/return`; it can only be set at app creation through
     the MCP);
   - whether to set up **webhooks** now, and if so the public https base URL of the
     backend (production domain or a tunnel).

## 1. Connect the MCP

Check whether the `fleeex` MCP tools are available (`list_apps`). If not, follow the
**fleeex-connect-mcp** skill. Authentication is interactive (browser): hand it to the user
and wait.

No MCP possible? Fall back to the dashboard (`https://app.fleeex.dev`): the user creates
the app and keys there and pastes them into the env file themselves.

## 2. App and keys

Follow **fleeex-api-keys**:

1. `list_apps`. Reuse the app whose name matches; otherwise `create_app` with the name and
   `redirectUris` from step 0. It returns the first **live** key, shown once.
2. `create_api_key` with `mode: "sandbox"` for development.
3. Write both straight into the gitignored env file; never print them in chat.

## 3. SDK

Follow **fleeex-sdk**: install, create the single fleeex module, route existing LLM calls
through it, pick a callable model, handle `402`, add the connection gate.

## 4. Webhooks (optional)

Follow **fleeex-webhooks**: receiver route with signature verification and dedup, then
`set_webhook` once a public https URL exists.

## 5. Verify

1. Typecheck, lint and run the project's tests.
2. Run one real call with the **sandbox** key (see **fleeex-sandbox**): expect a canned
   answer and the `x-fleeex-mode: sandbox` response header. It costs nothing.
3. If the user wants, exercise the `402` path in sandbox.

## 6. Report

Keep it short:

- app name and `appId`, key ids and modes (never key values);
- env variables written and the file they are in;
- what the user still has to do by hand, typically:
  - set `FLEEEX_API_KEY` to the **live** key in the production secret store (it is in
    `FLEEEX_LIVE_API_KEY` locally);
  - register the webhook once deployed, if it was deferred;
  - add redirect URIs later from the dashboard, if needed.

## Rules

- Never put a fleeex key in client code or in a public env prefix (`NEXT_PUBLIC_`,
  `VITE_`, `EXPO_PUBLIC_`, `PUBLIC_`, `REACT_APP_`).
- `userId` comes from the server-side authenticated session, never from a request body,
  query or header the client controls.
- Destructive MCP tools (`revoke_api_key`, `rotate_webhook_secret`) need the user's
  explicit approval first.
- No MCP tool moves money. Funding, spend caps and disconnecting apps belong to the end
  user in the fleeex dashboard.
